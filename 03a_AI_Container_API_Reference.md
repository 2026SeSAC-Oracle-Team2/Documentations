# 백엔드 ↔ AI 컨테이너 API 명세서 (Spring Boot ↔ FastAPI — 실구현 기준)

> **버전:** v1.0 (2026-09-04) — **실구현 DTO 역추적** (demo 브랜치, `dto/aicontainer/AiContainerDtos.kt` + `ai/AiContainerClient.kt` 실측)
> **용도:** AI 컨테이너(FastAPI) 구현자가 그대로 따라 할 수 있는 요청/응답 JSON 예시 집합.
> 설계 계약 = `03_AI_Container_Contract.md` · 전환 가이드 = `05a_Client_API_Reference.md` §6
> **JSON 키 네이밍은 본 문서가 단일 기준** — 백엔드 DTO의 `@JsonProperty` 그대로. 컨테이너는 이 키를 정확히 지켜야 함.

## 0. 전제

| 항목 | 내용 |
|------|------|
| 호출 방향 | **Spring Boot → AI 컨테이너 (FastAPI)** 단방향. 컨테이너는 요청받아 응답만 |
| 텍스트 | HTTP POST, `application/json` |
| 음성 파일 | **HTTP 전송 없음** — Docker volume 공유폴더로 교환 (m4a/mp3). 요청의 `userVoicePath`/응답의 `ttsPath`는 **공유폴더 기준 경로 문자열** |
| 영구 저장 | OCI는 백엔드 전담 — 컨테이너는 공유폴더에만 쓰고 버림 |
| 정합성 | 모든 요청/응답에 `sessionID` + `userID` 포함 — 백엔드가 대조 검증 |
| `turnId` | **컨테이너 로컬 번호 1~8** (DB TURN PK 아님, ADR-006). 백엔드가 응답 수신 후 실제 PK로 리매핑 |
| 에러/타임아웃 | 미정 (고도화 과제). 데모 단계는 항상 200 + 정상 바디 가정 |
| 스텁 전환 | 백엔드 `ai.container.mode: stub\|real` — 컨테이너는 real 모드에서 아래 전체를 구현 |

## 1. 공통 객체

### 1.1 userInfos (유저 정보 — 맥락용)

```json
{ "nickname": "노녹이", "likes": "카페, 산책", "sex": "M", "age": 27 }
```

| 키 | 타입 | nullable | 비고 |
|----|------|----------|------|
| nickname | String | X | |
| likes | String | O | 관심사 태그 (자유 텍스트) |
| sex | String | O | |
| age | Int | O | |

### 1.2 userVoiceEval (유저 음성 평가 — 채점 응답 공통)

```json
{
  "durationSecond": 7,
  "syllables": 12,
  "speakingTime": 2.4,
  "articulationTime": 1.8,
  "text": "포도!"
}
```

| 키 | 타입 | 비고 | DB 적재처 |
|----|------|------|-----------|
| durationSecond | Int | 녹음 전체 길이(초) | VOICE_RECORD.DURATION_SECONDS |
| syllables | Int | 음절 수 (발화지표 1) | VOICE_RECORD.SYLLABLES |
| speakingTime | Decimal | 발화시간 (발화지표 2) | VOICE_RECORD.SPEAKING_TIME |
| articulationTime | Decimal | 조음시간 (발화지표 3) | VOICE_RECORD.ARTICULATION_TIME |
| text | String | 유저 발화 STT | TURN.ANSWER_TEXT |

### 1.3 turnResult (턴 결과 — aichat/report에서 재사용)

```json
{ "turnId": 3, "type": "naming", "context": "포도", "userAnswer": "포도!", "score": 82.0 }
```

| 키 | 타입 | 비고 |
|----|------|------|
| turnId | Int | 컨테이너 로컬 번호 |
| type | String | 소문자: listen / naming / shadowing / selfTalk |
| context | String? | 문제 context — LISTEN: 지문 / NAMING: 정답단어 / SHADOWING: 원문 / SELF_TALK: 이미지 이름 |
| userAnswer | String? | 유저 답변 (STT 텍스트). LISTEN은 선택 결과 |
| score | BigDecimal? | 평가지표 점수 |

### 1.4 chatMessage (대화 로그 — aichat context / report talkContext)

```json
{ "speaker": "AI", "text": "아까 문제 푸느라 고생했네요! 오늘 하루 어땐어요?" }
{ "speaker": "USER", "text": "오늘은 카페에 갔어요" }
```

## 2. POST /sessions — 문제 일괄 생성

세션 생성 시 백엔드가 1회 호출. 8문제(4타입 × 각 2회, 무작위 순서)를 한 번에 생성. 스텁 지연 2~3초.

### 요청

```json
{
  "sessionId": 101,
  "thema": "TEST",
  "imageList": [
    { "imageId": 88, "imageName": "사과" },
    { "imageId": 89, "imageName": "포도" },
    { "imageId": 90, "imageName": "오렌지" },
    { "imageId": 92, "imageName": "딸기" },
    { "imageId": 93, "imageName": "감자" }
  ],
  "userID": 1,
  "userInfos": { "nickname": "노녹이", "likes": "카페, 산책", "sex": "M", "age": 27 },
  "userAQ": 72
}
```

| 키 | 타입 | 비고 |
|----|------|------|
| sessionId | Long | 백엔드 발급 세션 ID |
| thema | String | TEST / HOSPITAL / CAFE |
| imageList | Array | **백엔드가 조건 필터한 풀** — NAMING용(SEMANTIC_CUE 보유)과 SELF_TALK용(IMAGE_TAG_PATH 보유) 모두 포함되어 있으나 컨테이너는 **B-3 이후 필터 보장** 받음. 어떤 이미지를 어느 문제에 쓸지 선택은 컨테이너(LLM) 몫 |
| namingImageIds | Array&lt;Long&gt;? | **(B-3 신규, 선택)** NAMING에 사용 가능한 이미지 id 부분집합 — SEMANTIC_CUE 보유 이미지. 조건 풀이 최소 개수(2개) 미달 시 백엔드가 완화해 전체 풀 id를 보냄(로그 경고). 미수신(null) 시 imageList 전체에서 LLM 자율 선택 — 기존 계약 유지 |
| selfTalkImageIds | Array&lt;Long&gt;? | **(B-3 신규, 선택)** SELF_TALK에 사용 가능한 이미지 id 부분집합 — IMAGE_TAG_PATH 보유 이미지. 완화 규칙 동일 |
| userID | Long | ⚠️ `userID` — ID 대문자 |
| userInfos | object | §1.1 |
| userAQ | Int? | 유저 대표 AQ (최근 20세션 AQ 상위 10 평균). 세션 0개면 **null** |

### 응답

```json
{
  "sessionId": 101,
  "userID": 1,
  "problemList": [
    {
      "turnId": 1, "type": "listen",
      "ttsPath": "{userUUID}/101/1_ai.mp3",
      "passage": "사과를 고르세요",
      "perType": {
        "correct": 2,
        "options": [
          { "type": "text", "context": "8월 12일" },
          { "type": "image", "context": "88" },
          { "type": "text", "context": "8월 20일" }
        ]
      }
    },
    {
      "turnId": 2, "type": "naming",
      "ttsPath": "{userUUID}/101/2_ai.mp3",
      "passage": "이것은 무엇일까요? 사진 속 물건의 이름을 말해보세요.",
      "perType": { "correct": "포도" }
    },
    {
      "turnId": 3, "type": "shadowing",
      "ttsPath": "{userUUID}/101/3_ai.mp3",
      "passage": "나는 오늘 아침에 병원에 갔습니다",
      "perType": null
    },
    {
      "turnId": 4, "type": "selfTalk",
      "ttsPath": "{userUUID}/101/4_ai.mp3",
      "passage": "다음 상황을 보고 묘사해보세요",
      "perType": { "image": 93 }
    }
    // 총 8개 (각 타입 2회) — 무작위 순서
  ]
}
```

| 키 | 타입 | 비고 |
|----|------|------|
| sessionId / userID | Long | 요청 값 그대로 (정합성) |
| problemList[].turnId | Int | **로컬 번호 1~8** — DB PK 아님 |
| problemList[].type | String | **소문자**: listen / naming / shadowing / selfTalk |
| problemList[].ttsPath | String | 공유폴더상 AI TTS 경로: `{userUUID}/{sessionID}/{로컬turnId}_ai.mp3`. 컨테이너가 이 경로에 mp3 파일을 **실제로 생성**해야 함. (스텁은 `stub/tts_{n}_ai.mp3` 더미 문자열 반환 — 실컨테이너는 실경로 + 실파일) |
| problemList[].passage | String | TTS가 읽은 텍스트 = TURN.prompt_text |
| perType | object? | listen: `correct`(정답 options 인덱스, Int) + `options` / naming: `correct`(정답 단어) / shadowing: **null** (지문=문제) / selfTalk: `image`(이미지 id) |

**options 규약 (LISTEN):** 2~4개, 텍스트/이미지 혼합 가능. `type`: `text`(context=표시 텍스트) 또는 `image`(context=image_id 문자열). `correct`는 options 배열에서 정답의 **인덱스**.

> LISTEN 채점은 컨테이너 호출 없음 — 백엔드가 correct만 받아 자체 채점(100/0). `/answer/listen`은 **존재하지 않음**.

## 3. POST /answer/naming — 이름대기 채점

스텁 지연 0.8~1.5초. 감점 규칙(hintCount 반영)은 컨테이너 내부 루브릭.

### 요청

```json
{
  "sessionID": 101,
  "userID": 1,
  "problemContext": "포도",
  "userVoicePath": "shared/1/101/502_user.m4a",
  "hintCount": 1,
  "userRT": 2.1
}
```

| 키 | 타입 | nullable | 비고 |
|----|------|----------|------|
| sessionID / userID | Long | X | ⚠️ `sessionID` — ID 대문자 |
| problemContext | String | X | 정답 단어 (이미지 이름) |
| userVoicePath | String | X | 공유폴더상 유저 음성 경로 |
| hintCount | Int | X | 0~2 (의미→조음 순 공개 수) |
| userRT | BigDecimal | **O** | 유저 평균 RT — 최근 naming 음성 20개 중 발화시간/음절수 최단 10개의 (발화시간 총합 ÷ 음절수 총합). 백엔드 산정. **0개면 null** |

### 응답

```json
{
  "sessionID": 101,
  "userID": 1,
  "scoreNaming": 82.0,
  "userVoiceEval": {
    "durationSecond": 4,
    "syllables": 2,
    "speakingTime": 1.2,
    "articulationTime": 0.8,
    "text": "포도!"
  }
}
```

## 4. POST /answer/shadowing — 따라말하기 채점

### 요청

```json
{
  "sessionID": 101,
  "userID": 1,
  "problemContext": "나는 오늘 아침에 병원에 갔습니다",
  "userVoicePath": "shared/1/101/503_user.m4a"
}
```

### 응답

```json
{
  "sessionID": 101,
  "userID": 1,
  "scoreShadowing": 86.0,
  "userVoiceEval": {
    "durationSecond": 6, "syllables": 15,
    "speakingTime": 3.1, "articulationTime": 2.5,
    "text": "나는 오늘 아침에 병원에 갔습니다"
  }
}
```

## 5. POST /answer/selfTalk — 자발화 채점

`problemTag`는 tags.json **파일 내용 통째로** — 백엔드가 OCI에서 읽어 문자열로 전달. 컨테이너는 내부 스키마를 해석(파싱)해 채점.

### 요청

```json
{
  "sessionID": 101,
  "userID": 1,
  "problemImage": "감자",
  "problemTag": "{ \"tags\": [\"뿌리 채소\", \"흙\", \"밭\", \"군자\", \"전분\"] }",
  "userVoicePath": "shared/1/101/504_user.m4a"
}
```

| 키 | 비고 |
|----|------|
| problemImage | 이미지 **이름** (LLM 맥락용 — image_id는 응답 problem에서 백엔드가 수신) |
| problemTag | tags.json 원문 문자열. JSON 내부 스키마는 LLM 팀 규약 — 컨테이너가 해석 |

### 응답

```json
{
  "sessionID": 101,
  "userID": 1,
  "scoreSelfTalk": 55.0,
  "userVoiceEval": { "durationSecond": 12, "syllables": 30, "speakingTime": 6.5, "articulationTime": 5.2, "text": "감자는 흙에서 나는 뿌리 채소예요" }
}
```

> ⚠️ 구 스텁의 `scoreShadowing` 키 복붙 실수는 이미 정정 — 실컨테이너는 **`scoreSelfTalk`** 키로 응답.

## 6. POST /aichat — AI 자유대화 (이야기 턴)

선생성 불가 — 턴마다 1회 호출. 스텁 지연 1~2초. **AI 음성(TTS) 없음** — 텍스트만.

### 요청

**첫 호출 (AI가 대화 개시 — 음성 없음):**

```json
{
  "sessionID": 101,
  "userID": 1,
  "userInfos": { "nickname": "노녹이", "likes": "카페, 산책", "sex": "M", "age": 27 },
  "turnResults": [
    { "turnId": 1, "type": "listen",   "context": "사과를 고르세요", "userAnswer": "사과", "score": 100.0 },
    { "turnId": 2, "type": "naming",   "context": "포도", "userAnswer": "포도!", "score": 82.0 },
    { "turnId": 3, "type": "shadowing","context": "나는 오늘 아침에 병원에 갔습니다", "userAnswer": "나는 오늘 아침에 병원에 갔습니다", "score": 86.0 },
    { "turnId": 4, "type": "selfTalk", "context": "감자", "userAnswer": "감자는 뿌리 채소예요", "score": 55.0 }
    // ... 8턴 전부
  ],
  "context": [],
  "userVoicePath": null
}
```

**이후 턴 (유저 발화 있음):**

```json
{
  "sessionID": 101,
  "userID": 1,
  "userInfos": { "..." : "..." },
  "turnResults": [ "..." ],
  "context": [
    { "speaker": "AI",   "text": "아까 문제 푸느라 고생했네요! 오늘 하루 어땐어요?" },
    { "speaker": "USER", "text": "오늘은 카페에 갔어요" }
  ],
  "userVoicePath": "shared/1/101/511_user.m4a"
}
```

| 키 | 타입 | 비고 |
|----|------|------|
| turnResults | Array | 8개 문제풀이 턴의 결과 — 첫 대사 맥락으로 사용 |
| context | Array | §1.4 chatMessage 배열. **첫 호출은 빈 배열** |
| userVoicePath | String? | 첫 호출은 **null** (multipart 없이 BE가 호출) |

### 응답

```json
{
  "sessionID": 101,
  "userID": 1,
  "llmResponse": "아까 문제 푸느라 고생했네요! 오늘 하루 어땐어요?",
  "userText": "오늘은 카페에 갔어요"
}
```

| 키 | 타입 | nullable | 비고 |
|----|------|----------|------|
| llmResponse | String | X | AI 발화 — TURN.prompt_text 적재 + 화면 표시 |
| userText | String? | **O** | 이번 턴 유저 발화 STT — TURN.answer_text 적재 + 유저 말풍선. **첫 호출 응답에는 null** (유저 발화 없음) |

> ⚠️ ~~스텁 현재 동작: userText가 항상 null~~ → **B-2 수정 완료 (2026-09-03):** 음성이 있는 턴(`userVoicePath != null`)은 스텁이 더미 STT 텍스트를 userText로 반환한다. **첫 호출(음성 없는 턴)은 규약대로 null 유지.** 실컨테이너는 **이번 턴 userVoicePath 음성의 STT 결과**를 userText로 반환할 것.

## 7. POST /report — 세션 보고서 생성

세션 종료(정상·조기 모두) 시점에 백엔드가 1회 호출. 스텁 지연 2~3초. **동기 응답** — 클라가 결과 화면에서 로딩 대기 중.

### 요청

```json
{
  "sessionID": 101,
  "userID": 1,
  "turns": [
    { "turnId": 1, "type": "listen",    "context": "사과를 고르세요", "userAnswer": "사과", "score": 100.0 },
    { "turnId": 2, "type": "naming",    "context": "포도", "userAnswer": "포도!", "score": 82.0 },
    { "turnId": 3, "type": "shadowing", "context": "나는 오늘 아침에 병원에 갔습니다", "userAnswer": "...", "score": 86.0 },
    { "turnId": 4, "type": "selfTalk",  "context": "감자", "userAnswer": "...", "score": 55.0 },
    { "turnId": 5, "type": "storytelling", "context": null, "userAnswer": "오늘은 카페에 갔어요", "score": null }
    // 문제풀이 8턴 + 이야기 턴 전부
  ],
  "talkContext": [
    { "speaker": "AI", "text": "아까 문제 푸느라 고생했네요! 오늘 하루 어땐어요?" },
    { "speaker": "USER", "text": "오늘은 카페에 갔어요" },
    { "speaker": "AI", "text": "카페면 오늘 날씨가 좋았나 보네요. 뭐 드셨어요?" },
    { "speaker": "USER", "text": "아이스 아메리카노 마셨어요" }
  ]
}
```

| 키 | 비고 |
|----|------|
| turns | turnResult 배열 (§1.3) — 문제풀이 턴 + 이야기 턴 전부 포함. 이야기 턴은 type=`storytelling`, score=null |
| talkContext | chatMessage 배열 — 이야기 턴 전체 대화 로그 |

### 응답

```json
{
  "sessionID": 101,
  "userID": 1,
  "sessionAQ": 67,
  "sessionFeedbacks": {
    "listenFeedback": "알아듣기 문제를 대부분 정확히 골랐어요.",
    "namingFeedback": "힌트를 사용하면 점수가 내려가요. 처음 떠오른 이름을 그대로 말해보는 연습이 필요해요.",
    "shadowingFeedback": "문장을 또박또박 따라했어요.",
    "selfTalkFeedback": "상황 묘사에 핵심 단어가 일부 빠졌어요.",
    "talkFeedback": "자연스럽게 대화를 이어갔어요.",
    "totalFeedback": "전반적으로 좋은 흐름이었어요. 특히 듣기가 좋았어요!"
  }
}
```

| 키 | 타입 | 비고 |
|----|------|------|
| sessionAQ | Int | **100점 만점 정수** — 소수점 **올림은 컨테이너 책임** (예: 66.x → 67). DB `LEARNING_SESSION.AQ` NUMBER(3) CHECK(0~100) 적재 |
| sessionFeedbacks | object | 피드백 6종 — DB 피드백 6컬럼에 각각 적재. 컨테이너 내부 로직이므로 전부 **non-null**로 반환할 것 (데이터 없으면 빈 문자열 또는 안내 문구) |

## 8. 음성 파일 경로 규약 (공유폴더)

| 파일 | 경로 | 생성 주체 |
|------|------|-----------|
| AI TTS mp3 | `{userUUID}/{sessionID}/{로컬turnId}_ai.mp3` | **컨테이너** — `/sessions` 처리 중 생성, ttsPath로 경로 반환 |
| 유저 발화 m4a | `{userUUID}/{sessionID}/{실제turnId}_user.m4a` | **백엔드** — 클라 업로드 시 공유폴더 사본 생성 + OCI 적재 |

- 백엔드는 `/sessions` 응답 수신 후 공유폴더의 `{로컬turnId}_ai.mp3`를 **실제 TURN PK**로 리네임해 OCI 적재 (ADR-006)
- 세션 종료/폐기 시 공유폴더 정리는 백엔드 담당
- ⚠️ 스텁 모드는 파일이 실제로 없음 — `stub/tts_{n}_ai.mp3` 더미 경로 반환, 스트리밍도 샘플 mp3로 대체

## 9. 스텁↔실컨테이너 차이 요약 (컨테이너 구현자 참고)

| 항목 | 스텁 (현재) | 실컨테이너 (구현 목표) |
|------|-------------|------------------------|
| 응답 지연 | Thread.sleep 2~3s 등 시뮬레이션 | 실제 추론 시간 (LLM=Ollama Cloud, STT=로컬 Whisper, TTS=로컬 Qwen) |
| ttsPath | `stub/tts_{n}_ai.mp3` 더미 | 공유폴더 실경로 + 실제 mp3 생성 |
| NAMING 정답 | imageList의 imageName 그대로 | 컨테이너가 이미지 선택 + 정답 단어 결정 |
| userText | null (B-2에서 더미 예정) → **✅ B-2 완료: 음성 턴 더미 STT 반환, 첫 호출 null 유지** | 이번 턴 음성의 실제 STT 결과 |
| 점수 | 유사도 시뮬레이션 60~95 + 감점 | 실제 채점 알고리즘 (루브릭 컨테이너 내부) |
| 리포트 | 고정 문구 + AQ 55~85 | 실제 AQ 산정 + 개인화 피드백 |
| 이미지 선택 | 풀에서 무작위 → **✅ B-3 완료: namingImageIds/selfTalkImageIds 부분집합 내에서 무작위 (완화 시 전체 풀)** | 유저 정보·맥락 고려 LLM 선택 (타입별 id 부분집합 수신 시 존중 권장) |

## 10. 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-09-04 | 초안 — demo 브랜치 실구현 DTO(`AiContainerDtos.kt`) 역추적. 엔드포인트별 전체 JSON 예시 포함. 스텁/실컨테이너 차이 표 (§9) |
| v1.1 | 2026-09-03 | B-2/B-3 반영 — §2 요청에 namingImageIds/selfTalkImageIds 선택 필드 추가(백엔드 조건 필터+완화 규약), §6 스텁 userText 더미 STT 수정 완료 표기, §9 차이표 갱신 |