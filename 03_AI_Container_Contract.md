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

## 2. POST /sessions/today · POST /sessions/theme — 세션 문제 일괄 생성 (v1.2 분기)

"오늘의 학습"/"테마별 학습" 세션 생성 시점. 평가지표 8턴(4종 × 각 2회)의 문제를 **한 번에 전부 생성**한다. **요청·응답 바디 필드는 두 엔드포인트가 완전히 동일**하고, 컨테이너 내부 출제 로직만 다르다.

| 엔드포인트 | 트리거 | 출제 로직 |
|-----------|--------|----------|
| `POST /sessions/today` | "오늘의 학습" → 백엔드 테마 **무작위 선택** | 테마 범위 내 8문제 **순서·내용 모두 무작위** |
| `POST /sessions/theme` | "테마별 학습" 테마 선택 → 백엔드 테마 **지정** | **기획 시나리오 플로우** 그대로 8문제 + AI 대화 첫 턴 맥락 |

### REQUEST BODY

| 변수명 | 설명 | 비고 |
|--------|------|------|
| `sessionId` | 세션 ID | 정합성 확인용 |
| `thema` | 테마 | `TEST` / `HOSPITAL` / `CAFE` |
| `imageListListening` | LISTEN용 이미지 풀 | **IMAGE_TAG_PATH 없고 SEMANTIC_CUE 없는 이미지** `[{imageId, imageName}]` |
| `imageListNaming` | NAMING용 이미지 풀 | **SEMANTIC_CUE 보유 이미지** (정답 단어 소스) |
| `imageListSelfTalk` | SELF_TALK용 이미지 풀 | **IMAGE_TAG_PATH 보유 + SEMANTIC_CUE 없는 이미지** (태그 채점용) |
| `userID` | 유저 ID | |
| `userInfos` | 유저 정보 | 아래 표 |
| `userAQ` | 유저 대표 AQ | 최근 20세션 중 AQ 상위 10개 평균 (정수). 세션 0개면 `null` — 추후 가입 설문으로 임시 AQ 주입 예정 |

> **이미지 풀 분할 규약 (v1.2):** 백엔드가 IMAGE_RESOURCE 데이터 기준으로 3개 풀로 분할 — 컨테이너는 각 타입 문제에 **해당 배열 내에서만** 이미지 선택. 분류 규칙: `IMAGE_TAG_PATH 있음 = SELF_TALK` / `TAG_PATH 없음 + SEMANTIC_CUE 있음 = NAMING(+LISTEN 공용)` / `TAG_PATH 없음 + CUE 없음 = LISTEN`. NAMING/LISTEN 이미지는 LISTEN 선택지로 재사용 가능. 풀 부족 시 백엔드가 완화(전체 풀 병합+경고). **(구 v1.1의 namingImageIds/selfTalkImageIds는 폐지 — 3분할이 대체)** JSON 예시는 `03a_AI_Container_API_Reference.md` §2 참조.

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

## 9. 리포트 생성 — 2단계 분리 (v1.2)

> **UX 개선 (2026-09-04 합의):** 리포트를 2단계로 분리 — 문제 8턴 종료 시점에 간이 보고서(AQ+4지표 피드백)를 먼저 확보해 이야기 턴 동안 백그라운드 완성 → 종료 후 즉시 간이 보고서 표시, 상세 보고서는 완료 후 앱 내 알림으로 수령. JSON 예시는 `03a_AI_Container_API_Reference.md` §7 참조.

### 9.1 POST /report/problems — 간이 보고서

**트리거:** 8번째 문제 답안 제출 시점 — 백엔드가 마지막 채점과 동시에 **자동 호출**. 이야기 턴이 진행되는 동안 컨테이너가 생성 완료. 스텁 지연 2~3초.

| 구분 | 내용 |
|------|------|
| 요청 | `sessionID`, `userID`, `turns`(문제풀이 8턴만 — 이야기 턴 미포함) |
| 응답 | `sessionAQ`(8문제 점수만으로 산출, 정수·올림은 컨테이너 책임) + `sessionFeedbacks` 6종 중 **listen/naming/shadowing/selfTalkFeedback만 non-null** (talk/total은 null) |
| DB 적재 | `LEARNING_SESSION.AQ` + 피드백 4컬럼 즉시 UPDATE |

### 9.2 POST /report/total — 상세 보고서

**트리거:** AI 대화 종료(정상·하드캡) 또는 이야기 중 조기종료 시점. 응답 대기 없이 백그라운드 생성 → 완료 후 앱 내 알림. 스텁 지연 **10초**.

| 구분 | 내용 |
|------|------|
| 요청 | `sessionID`, `userID`, `turns`(문제풀이 8턴), `talkContext`(이야기 전체 대화 로그 — 조기종료 시 빈 배열) |
| 응답 | `sessionFeedbacks` 중 **talkFeedback/totalFeedback만 non-null** (나머지 null — 백엔드는 2컬럼만 UPDATE) |
| DB 적재 | `LEARNING_SESSION` talk/total 피드백 2컬럼 부분 UPDATE |
| 상세 조회 추적 | `LEARNING_SESSION.REPORT_VIEWED_AT` — 클라가 상세 보고서 조회한 시점 기록 (null=미조회 → 앱 내 알림함/네비 버블 판별) |

### 9.3 조기종료(이야기 턴 없이 종료) 규약

- 세션 STATUS 전용 값: `COMPLETED_NO_TALK`
- `/report/total`은 **호출하지 않음** — talkFeedback/totalFeedback은 NULL 유지
- 클라 표시: "AI 대화를 진행하지 않아 대화 피드백이 없어요"

### 9.4 스텁 지연값

| 엔드포인트 | 스텁 지연 |
|-----------|-----------|
| /report/problems | 2~3초 (이야기 턴 진행 중 완료) |
| /report/total | **10초** (실컨테이너 추론 시간 감안) |

---

## 10. 산정식 (백엔드 계산 책임)

| 값 | 산정식 | 전달 위치 |
|----|--------|-----------|
| `userAQ` | 최근 20개 세션 중 AQ 상위 10개 세션의 AQ 평균. 세션 0개 → `null` | `POST /sessions/today` · `/sessions/theme` |
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
| v1.2 | 2026-09-04 | **컨테이너 협의 반영 (1):** §2 엔드포인트 분기 — `/sessions/today`(테마 랜덤+무작위 출제) / `/sessions/theme`(기획 시나리오 플로우), 요청·응답 바디 공통. **imageList 3분할** — imageListListening/Naming/SelfTalk(분류: TAG_PATH 있음=SELF_TALK, TAG 없음+CUE 있음=NAMING(+LISTEN 공용), 둘 다 없음=LISTEN), 구 namingImageIds/selfTalkImageIds 폐지. §9 리포트 2단계 분리 — /report/problems(간이: AQ+4지표 피드백, 8턴 종료 시 자동 호출) / /report/total(상세: talk+total 피드백, 종료 시 백그라운드), 조기종료 STATUS=COMPLETED_NO_TALK 규약, REPORT_VIEWED_AT 상세조회 추적. |