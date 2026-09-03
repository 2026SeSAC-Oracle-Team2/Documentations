# AI 컨테이너 API 계약서 (Spring Boot ↔ AI Container)

> **버전:** v1.0 (2026-09-02 기획 확정)
> **상위 문서:** `06_Session_Flow_Spec.md` (세션/기획), `04_Database_Design.md` (스키마)
> **전제:** Spring Boot 서버가 AI 컨테이너의 FastAPI로 HTTP 요청을 보낸다.
> AI 컨테이너는 통합 컨테이너로 **STT / LLM 텍스트 생성 / TTS 생성 / 채점 / 리포트**를 전부 담당한다. (구 llm-container + scoring-container 통합, ADR-002)

## 0. 통신 및 파일 교환 원칙

| 구분 | 방식 | 대상 |
|------|------|------|
| **텍스트 데이터** | HTTP POST (JSON 바디) | 태그, 대화 컨텍스트, 문제 정보, 채점 결과 등 |
| **음성 파일 (m4a/mp3)** | **Docker Compose volume 마운트 공유폴더** | 유저 발화 m4a, AI TTS mp3 — HTTP 전송하지 않음 |
| **영구 저장** | OCI Object Storage | 공유폴더는 처리용 임시 저장소. 백엔드가 OCI에 적재 (ADR-005) |
| **클라이언트 서빙** | 백엔드 프록시 스트리밍 | 클라이언트 ↔ 백엔드는 항상 API, 컨테이너 직접 접근 없음 |

**모델 구성 (컨테이너 내부):** LLM = Ollama Cloud API / STT = 로컬 Whisper / TTS = 로컬 Qwen-TTS

> ⚠️ **동시성 로드맵:** 현재 컨테이너 1개, 직렬 처리. 동시 접속 대응 시 컨테이너 다중 기동 + Redis + Celery 메시지 큐 오케스트레이션 도입 예정. 로컬 모델(STT/TTS)이 병목 시 클라우드 모델로 교체 가능하도록 wrapper 구조 유지.

## 1. 공통 규약

| 항목 | 규약 |
|------|------|
| 식별자 정합성 | 모든 요청/응답에 `sessionId`, `userId`를 포함 — 백엔드가 정합성 검증 |
| 문제 타입 표기 | API/JSON은 소문자 camelCase (`listen`, `naming`, `shadowing`, `selfTalk`, `storytelling`). DB 저장 시 백엔드가 대문자 seed 코드(`LISTEN` 등)로 매핑 |
| `turnId` | 컨테이너가 부여하는 **로컬 번호 1~8**. DB의 TURN PK와 별개 — 백엔드가 응답 수신 시 실제 TURN PK로 리매핑 (ADR-006) |
| `ttsPath`의 턴 ID | `{턴ID}_ai.mp3`의 턴ID도 컨테이너 로컬 번호. 백엔드가 실제 TURN PK로 리네임 후 OCI 적재 |
| userInfos | `{nickname, likes?, sex?, age?}` — likes/sex/age는 nullable (추후 확장) |
| 에러/타임아웃 | **미정 (고도화 과제).** 데모 단계에서는 규약 없음 — 컨테이너 가용성 가정 |

---

## 2. POST /sessions — 세션 문제 일괄 생성

"오늘의 학습" 세션 생성 시점. 평가지표 8턴(4종 × 각 2회, 무작위 순서)의 문제를 **한 번에 전부 생성**한다.

### REQUEST BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionId` | 세션 ID | 정합성 확인용 |
| `thema` | 테마 | 서버가 랜덤 선택한 값 (`TEST` / `HOSPITAL` / `CAFE`) |
| `imageList` | 선택된 테마의 이미지 리스트 | `[{imageId, imageName}]` — IMAGE_THEMA로 조회한 테마 이미지 풀. **어떤 이미지를 어느 문제에 쓸지는 AI(LLM)가 선택**. ⚠️ **조건 필터는 백엔드 책임 (B-3, 2026-09-03):** NAMING용은 SEMANTIC_CUE 보유 이미지, SELF_TALK용은 IMAGE_TAG_PATH 보유 이미지로 필터해 별도 필드(`namingImageIds`/`selfTalkImageIds`, 선택)로 동봉 — 조건 풀 부족 시 백엔드가 완화+경고. 스텁은 부분집합을 존중하며, 실컨테이너도 수신 시 존중 권장 |
| `userID` | 유저 ID | |
| `userInfos` | 유저 정보 | 아래 표 |
| `userAQ` | 유저 대표 AQ | 최근 20세션 중 AQ 상위 10개 평균 (정수). 세션 0개면 `null` — 추후 가입 설문으로 임시 AQ 주입 예정 |

#### userInfos

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `nickname` | 닉네임 | |
| `likes` | 관심사 태그 | nullable |
| `sex` | 성별 | nullable |
| `age` | 나이 | nullable |

### RESPONSE BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionId` | 세션 ID | 정합성 확인용 |
| `userID` | 유저 ID | 정합성 확인용 |
| `problemList` | 문제 리스트 | 8개 — 무작위 순서 |

#### problem

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `turnId` | **컨테이너 로컬 번호 (1~8)** | DB TURN PK 아님. 백엔드가 리매핑 |
| `type` | 문제 타입 | `listen` / `naming` / `shadowing` / `selfTalk` |
| `ttsPath` | AI TTS 음성 경로 | 공유폴더상 `{유저UUID}/{세션ID}/{로컬턴ID}_ai.mp3` |
| `passage` | 지문 | TTS가 읽은 텍스트 → `TURN.prompt_text` |
| `perType` | 타입별 데이터 | 아래 참조 |

#### perType — listen (알아듣기)

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `correct` | 정답 | **선택지 중 정답의 인덱스** |
| `options` | 선택지 | 2~4개. 텍스트형/이미지형 혼합 가능 |

```json
// 텍스트 선택지형
"options": [
  { "type": "text",  "context": "8월 12일" },
  { "type": "text",  "context": "8월 20일" }
]
// 그림 선택지형
"options": [
  { "type": "image", "context": "image_id" },
  { "type": "image", "context": "image_id" }
]
```

> LISTEN 채점은 컨테이너 호출 없이 **백엔드 자체 처리** (정답 100 / 오답 0, ADR-003).

#### perType — naming (이름대기)

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `correct` | 정답 단어 | 이미지 이름 → `TURN.correct_value` |

#### perType — shadowing (따라말하기)

지문 == 문제라서 `perType` 없음.

#### perType — selfTalk (자발화)

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `image` | 문제 상황 이미지 id | → TURN_IMAGE 매핑 |

---

## 3. ~~POST /answer/listen~~ — 폐지

LISTEN은 정답 인덱스가 `/sessions` 응답으로 이미 백엔드가 보유하므로 컨테이너 호출 불필요. 유저가 화면에서 선택지 탭 → 백엔드가 대조 → 점수 산정 (정답 100 / 오답 0). 유저 음성·발화지표 없음.

---

## 4. POST /answer/naming (이름대기 채점)

> 예시: [마우스 사진] → 유저 발화 "마..마우스!"

### REQUEST BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionID` | 세션 ID | 정합성용 |
| `userID` | 유저 ID | 정합성용 |
| `problemContext` | 정답 확인용 | 정답 단어 (이미지 이름, 예: "딸기") |
| `userVoicePath` | 유저 음성 경로 | **공유폴더상 경로** |
| `hintCount` | 힌트 사용 횟수 | 0~2 (의미단서→조음단서 순). DB `HINTS_SHOWN`과 동일 개념 |
| `userRT` | 유저 평균 RT | 수식: 최근 naming 유형 20개 중 `발화시간/음절수`가 가장 짧았던 10개의 `발화시간 총합 / 음절수 총합`. **백엔드가 계산해 전달.** 대상 0개면 `null` |

### RESPONSE BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionID` | 세션 ID | 정합성용 |
| `userID` | 유저 ID | 정합성용 |
| `scoreNaming` | 채점 결과 점수 | RT, 힌트 사용횟수 등 반영. 루브릭은 컨테이너 내부 구현 |
| `userVoiceEval` | 유저 음성 평가 객체 | 아래 공통 객체 |

## 5. POST /answer/shadowing (따라말하기 채점)

### REQUEST BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionID` | 세션 ID | 정합성용 |
| `userID` | 유저 ID | 정합성용 |
| `problemContext` | 정답 확인용 | 원문 구문 (예: "나는 장풍을 했다") |
| `userVoicePath` | 유저 음성 경로 | 공유폴더상 경로 |

### RESPONSE BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionID` / `userID` | 정합성용 | |
| `scoreShadowing` | 채점 결과 | |
| `userVoiceEval` | 유저 음성 평가 객체 | |

## 6. POST /answer/selfTalk (자발화 채점)

### REQUEST BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionID` | 세션 ID | 정합성용 |
| `userID` | 유저 ID | 정합성용 |
| `problemImage` | 이미지 이름 | LLM이 이름 기준으로 맥락 구성 (image_id는 응답 시 수신 가능) |
| `problemTag` | 정답 확인용 태그 | `tags.json` **파일 내용 통째로** (백엔드가 OCI에서 읽어 전달, 파싱 불필요) |
| `userVoicePath` | 유저 음성 경로 | 공유폴더상 경로 |

### RESPONSE BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionID` / `userID` | 정합성용 | |
| `scoreSelfTalk` | 채점 결과 | 태그 커버리지 기반 |
| `userVoiceEval` | 유저 음성 평가 객체 | |

## 7. userVoiceEval — 공통 유저 음성 평가 객체

마이크가 들어가는 모든 `/answer/*` 응답에 포함. → `VOICE_RECORD` USER 행 적재 (발화지표 3종).

| 변수명 | 설명 | DB 매핑 |
|--------|------|---------|
| `durationSecond` | 녹음 시간 | `DURATION_SECONDS` |
| `syllables` | 음절 수 | `SYLLABLES` |
| `speakingTime` | 발화시간 | `SPEAKING_TIME` |
| `articulationTime` | 조음시간 | `ARTICULATION_TIME` |
| `text` | 유저 발화 텍스트 (STT) | `TURN.answer_text` |

> **용어:** 발화지표 = `SYLLABLES` / `SPEAKING_TIME` / `ARTICULATION_TIME` 3종. 지표의 측정·정의는 컨테이너 내부 구현 기준 — 백엔드는 수신 적재만.

---

## 8. POST /aichat — AI 자유대화 (STORYTELLING 턴)

이야기 턴은 선생성 불가 — **진행 중 한 턴씩 생성**. AI 음성(TTS)은 현재 사용하지 않음 (지연 우려, 향후 확장 대비 DB 구조 유지).

### REQUEST BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionID` | 세션 ID | 정합성용 |
| `userID` | 유저 ID | 정합성용 |
| `userInfos` | 유저 정보 | 맥락용 (§1 userInfos와 동일 구조) |
| `turnResults` | 8개 턴의 결과 | 맥락용 — 문제 유형, 문제 context, 사용자 답변, 점수 |
| `context` | 대화 맥락 | `[{"speaker": "AI"\|"USER", "text": "..."}]` 배열. **첫 요청은 빈 배열** — userInfos + turnResults 기반으로 AI가 첫 대사로 대화 개시 |
| `userVoicePath` | 유저 음성 경로 | 공유폴더상 경로. (첫 요청에는 없음 — AI가 먼저 말을 걸므로) |

### RESPONSE BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionID` / `userID` | 정합성용 | |
| `llmResponse` | LLM 응답 텍스트 | 화면 표시용 (TTS 미사용) |
| `userText` | **유저 발화 STT 텍스트** | `TURN.answer_text` 기록용 (첫 요청 응답에는 없음) |

> 이야기 턴: 채점 없음, 발화지표 없음. TURN에 `prompt_text` = AI 발화 / `answer_text` = 유저 발화(STT) 기록.

## 9. POST /report — 세션 보고서 생성

세션 종료 시점(정상·조기 종료 모두)에 백엔드가 호출. 응답 수신까지 결과 화면은 로딩 대기.

### REQUEST BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionID` | 세션 ID | 정합성용 |
| `userID` | 유저 ID | 정합성용 |
| `turns` | 세션에 포함된 턴들 정보 | turn = 문제 유형, 문제 context, 사용자 답변, 점수 |
| `talkContext` | AI 대화 맥락 | `/aichat`의 context와 동일 형식 (스피커/텍스트 배열) |

### RESPONSE BODY

| 변수명 | 설명 | DB 저장 |
|--------|------|---------|
| `sessionID` / `userID` | 정합성용 | — |
| `sessionAQ` | 세션 AQ (100점 만점, **정수 — 소수점 올림은 컨테이너 책임**) | `LEARNING_SESSION.AQ` |
| `sessionFeedbacks` | 피드백 6종 | `LEARNING_SESSION` 피드백 6컬럼 |

#### sessionFeedbacks

| 필드 | 설명 |
|------|------|
| `listenFeedback` | 알아듣기 지표 피드백 |
| `namingFeedback` | 이름대기 지표 피드백 |
| `shadowingFeedback` | 따라말하기 지표 피드백 |
| `selfTalkFeedback` | 스스로말하기 지표 피드백 |
| `talkFeedback` | AI 대화(STORYTELLING) 피드백 |
| `totalFeedback` | 종합 피드백 |

---

## 10. 산정식 (백엔드 계산 책임)

| 값 | 산정식 | 전달 위치 |
|----|--------|-----------|
| `userAQ` | 최근 20개 세션 중 AQ 상위 10개 세션의 AQ 평균. 세션 0개 → `null` | `POST /sessions` |
| `userRT` | 최근 NAMING 음성 20개 중 발화시간/음절수 최단 10개의 (발화시간 총합 ÷ 음절수 총합). 0개 → `null` | `/answer/naming` |

> 지표별 실력(대시보드)도 동일식: 최근 20세션 중 지표별 상위 10개 세션 평균 (ADR-009). AQ 계산 로직 자체는 컨테이너 구현 — 백엔드는 알 필요 없음.

## 11. 음성 파일 라이프사이클

| 단계 | 동작 |
|------|------|
| ① AI TTS 생성 | 컨테이너가 공유폴더에 `{유저UUID}/{세션ID}/{로컬턴ID}_ai.mp3` 생성 → `/sessions` 응답 |
| ② 백엔드 수신 시 | 공유폴더 파일을 실제 TURN PK로 리네임 → OCI 적재 (`containers/llm/{userUUID}/{sessionID}/{turnID}_ai.mp3`) |
| ③ 유저 발화 | 클라 multipart 업로드 → 백엔드가 즉시 OCI 적재 + 공유폴더 사본 생성 → `/answer/*`의 `userVoicePath`로 전달 |
| ④ 클라 재생 | 백엔드 프록시 스트리밍 (OCI 원본, JWT 인증) |
| ⑤ 정리 | 공유폴더 파일은 **세션 종료/폐기 시점에 삭제** (OCI는 영구 보존) |

## 12. 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-09-02 | 초안 확정 — 세션 기획 회의(T~Z 확정사항) 반영: /answer/listen 삭제(BE 자체채점), /aichat userText 추가, selfTalk scoreSelfTalk 오타 수정, turnId 리매핑 규약, 음성 라이프사이클 확정 |
| v1.1 | 2026-09-03 | §2 imageList에 B-3 조건 필터 책임 주석 갱신 — NAMING=SEMANTIC_CUE/SELF_TALK=IMAGE_TAG_PATH 필터는 백엔드가 수행, 선택 필드 namingImageIds/selfTalkImageIds로 전달(완화 규약 포함) |