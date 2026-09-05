# 백엔드 ↔ AI 컨테이너 API 명세서 (Spring Boot ↔ FastAPI — 실구현 기준)

> **버전:** v1.8 (2026-09-05) — D-4 userMemory 개인화 구현 완료 반영 (스텁 갱신 시뮬레이션 + 백엔드 라이프사이클 실측 기준, 협의분 중 미구현은 구현 예정 상태 표기 유지)
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
{
  "nickname": "노녹이",
  "hobbies": "가끔 낚시 갈 정도",
  "tags": "등산, 골프, 요리",
  "sex": "M",
  "age": 27,
  "userMemory": "..."
}
```

| 키 | 타입 | nullable | 비고 |
|----|------|----------|------|
| nickname | String | X | |
| hobbies | String | O | 취미 자유 텍스트 (구 `likes` — 가입 플로우 개편 v1.3) |
| tags | String | O | 관심사 태그 — TAGS 테이블에서 최대 5개 선택, 쉼표 구분 문자열로 통째 전달 (건강관리/등산/골프/여행/트로트/요리/텃밭가꾸기/낚시/독서/바둑/사진/전시관람/국내여행/반려동물/봉사활동) |
| sex | String | O | |
| age | Int | O | BIRTH_DATE 기반 백엔드 산정 (구 AGE 컬럼 — v1.3 폐지, 산정식 변경) |
| userMemory | String | O | **누적 개인화 메모리** (v1.3 신규) — AI 대화 기반으로 컨테이너가 관리하는 오파크 텍스트. 백엔드는 파싱·스키마 비관여, 통째로 전달/저장. 규약은 §10 참조 |

> **v1.3 (2026-09-04):** userInfos 개편 — `likes` → `hobbies` rename + `tags` 신설(선택 태그), `age`는 BIRTH_DATE 기반 산정값, `userMemory` 추가. 구 버전의 `likes` 키는 폐지.

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
| type | String | 소문자: listenText / listenPicture / naming / shadowing / selfTalk — **v1.4: LISTEN 세분화, 구 `listen` 폐지** |
| context | String? | 문제 context — LISTEN: 지문 / NAMING: 정답단어 / SHADOWING: 원문 / SELF_TALK: 이미지 이름 |
| userAnswer | String? | 유저 답변 (STT 텍스트). LISTEN은 선택 결과 |
| score | BigDecimal? | 평가지표 점수 |

### 1.4 chatMessage (대화 로그 — aichat context / report talkContext)

```json
{ "speaker": "AI", "text": "아까 문제 푸느라 고생했네요! 오늘 하루 어땐어요?" }
{ "speaker": "USER", "text": "오늘은 카페에 갔어요" }
```

## 2. POST /sessions/today · POST /sessions/theme — 문제 일괄 생성

세션 생성 시 백엔드가 1회 호출. **세션 종류에 따라 엔드포인트가 분기** — 요청/응답 필드는 동일하고 컨테이너 내부 출제 로직이 다름.

| 엔드포인트 | 트리거 | 출제 로직 (컨테이너 내부) |
|-----------|--------|--------------------------|
| `POST /sessions/today` | 유저가 "오늘의 학습" 선택 → 백엔드가 테마 **무작위 선택** | 지정 테마 범위 내에서 8문제 **순서·내용 모두 무작위** |
| `POST /sessions/theme` | 유저가 "테마별 학습"에서 특정 테마 선택 → 백엔드가 해당 테마 지정 | **기획 시나리오 플로우** 그대로 1~8문제 + AI 대화 첫 턴 맥락 출제 (무작위 아님) |

> 스텁 지연 2~3초. 문제풀이 8턴(4타입 × 각 2회). 아래 예시는 두 엔드포인트 **공통** — 바디 필드가 완전히 동일하므로 하나만 기술한다.

### 요청

```json
{
  "sessionId": 101,
  "thema": "TEST",
  "imageListListening": [
    { "imageId": 90, "imageName": "오렌지" }
  ],
  "imageListNaming": [
    { "imageId": 88, "imageName": "사과" },
    { "imageId": 89, "imageName": "포도" },
    { "imageId": 92, "imageName": "딸기" }
  ],
  "imageListSelfTalk": [
    { "imageId": 93, "imageName": "감자" }
  ],
  "userID": 1,
  "userInfos": { "nickname": "노녹이", "hobbies": "가끔 낚시 갈 정도", "tags": "등산, 골프", "sex": "M", "age": 27, "userMemory": "..." },
  "userAQ": 72
}
```

| 키 | 타입 | 비고 |
|----|------|------|
| sessionId | Long | 백엔드 발급 세션 ID |
| thema | String | TEST / HOSPITAL / CAFE |
| imageListListening | Array | **IMAGE_TAG_PATH 없고 SEMANTIC_CUE 없는 이미지** — LISTEN 전용 풀 |
| imageListNaming | Array | **SEMANTIC_CUE 보유 이미지** — NAMING 전용 풀 (정답 단어 소스) |
| imageListSelfTalk | Array | **IMAGE_TAG_PATH 보유 + SEMANTIC_CUE 없는 이미지** — SELF_TALK 전용 풀 (태그 채점용) |
| userID | Long | ⚠️ `userID` — ID 대문자 |
| userInfos | object | §1.1 |
| userAQ | Int? | 유저 대표 AQ — **USER_REPRESENTATIVE_SCORES.USER_AQ 캐시 조회 전달** (대표점수 전용 테이블 — 가입 설문 초기 세팅 30/70/90 → AQ 확정 시점마다 재계산 갱신, v1.7). 백엔드 산정식 실행 없음 — 캐시 직접 조회. null 발생 없음 (설문 강제). 컨테이너는 null 수신 대비 예외처리 유지 권장 |

**LISTEN 난이도 등급표 (v1.4 신설 — 03 계약서 §2.1 동일):**

| 등급 | AQ 점수 범위 | 이미지 태그 | 선택지 개수 |
|------|-------------|------------|------------|
| 1 | AQ < 60 | EASY | 2 |
| 2 | 60 ≤ AQ < 80 | EASY | 3 |
| 3 | 80 ≤ AQ < 94 | EASY | 4 |
| 4 | 94 ≤ AQ < 98 | HARD | 3 |
| 5 | AQ ≥ 98 | HARD | 4 |

> **v1.4 분담:** 백엔드 = userAQ → 등급 산정 → EASY/HARD 태그로 IMAGE_THEMA 조회 → imageListListening 필터링 전달. 컨테이너 = 등급에 맞는 선택지 개수로 options 구성 (등급표 컨테이너 보유 — 요청에 등급/개수 필드 없음). EASY=기존 태그(TEST/HOSPITAL/CAFE) / HARD=TEST_HARD·HOSPITAL_HARD·CAFE_HARD. EASY=사물 이미지 / HARD=행동·상황·사람 이미지.

> **이미지 풀 분할 규약 (v1.2, v1.4 갱신 — 구 imageList+namingImageIds 폐지):** 백엔드가 IMAGE_RESOURCE 데이터 기준으로 3개 풀로 분할해 전달 — 컨테이너는 각 타입 문제에 **해당 배열 내에서만** 이미지를 선택한다. 분류 규칙: `IMAGE_TAG_PATH 있음 = SELF_TALK` / `TAG_PATH 없음 + SEMANTIC_CUE 있음 = NAMING` / `TAG_PATH 없음 + CUE 없음 = LISTEN`. **v1.4: NAMING은 EASY 이미지 전용** (사물 — HARD의 행동·상황·사람 이미지는 NAMING 불가). NAMING 이미지의 LISTEN 선택지 재사용은 등급 필터 통과분(imageListListening 배열) 내로 한정. imageListListening은 **userAQ 등급 기반 EASY/HARD 태그 필터 결과**만 전달. **(구 v1.1의 namingImageIds/selfTalkImageIds 부분집합 필드는 폐지 — 3분할이 대체)** 풀 부족 시 백엔드가 완화 로직 적용(전체 풀에 병합 + 로그 경고).

### 응답

```json
{
  "sessionId": 101,
  "userID": 1,
  "problemList": [
    {
      "turnId": 1, "type": "listenPicture",
      "ttsPath": "{userUUID}/101/1_ai.mp3",
      "passage": "사과를 고르세요",
      "perType": {
        "correct": 2,
        "options": [
          { "type": "text", "context": "8월 12일" },
          { "type": "image", "context": "88" },
          { "type": "image", "context": "90" }
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
    {
      "turnId": 2, "type": "listenText",
      "ttsPath": "{userUUID}/101/2_ai.mp3",
      "passage": "오늘은 무슨 요일인가요?",
      "perType": {
        "correct": 0,
        "options": [
          { "type": "text", "context": "금요일" },
          { "type": "text", "context": "토요일" },
          { "type": "text", "context": "일요일" }
        ]
      }
    }
    // 이후 naming/shadowing/selfTalk 각 2회 — 총 8개 (LISTEN_TEXT·LISTEN_PICTURE 각 1회 포함), 무작위 순서
  ]
}
```

| 키 | 타입 | 비고 |
|----|------|------|
| sessionId / userID | Long | 요청 값 그대로 (정합성) |
| problemList[].turnId | Int | **로컬 번호 1~8** — DB PK 아님 |
| problemList[].type | String | **소문자**: listenText / listenPicture / naming / shadowing / selfTalk — **v1.4: LISTEN 세분화, 구 `listen` 폐지** |
| problemList[].ttsPath | String | 공유폴더상 AI TTS 경로: `{userUUID}/{sessionID}/{로컬turnId}_ai.mp3`. 컨테이너가 이 경로에 mp3 파일을 **실제로 생성**해야 함. (스텁은 `stub/tts_{n}_ai.mp3` 더미 문자열 반환 — 실컨테이너는 실경로 + 실파일) |
| problemList[].passage | String | TTS가 읽은 텍스트 = TURN.prompt_text |
| perType | object? | listenText·listenPicture: `correct`(정답 options 인덱스, Int) + `options` / naming: `correct`(정답 단어) / shadowing: **null** (지문=문제) / selfTalk: `image`(이미지 id) |

**options 규약 (LISTEN — v1.4):** listenText=텍스트형만 / listenPicture=이미지형만 (유형 고정, 혼합 폐지). 개수 2~4개 = 등급표(§2)에 따름. `type`: `text`(context=표시 텍스트) 또는 `image`(context=image_id 문자열). `correct`는 options 배열에서 정답의 **인덱스**.

> LISTEN(listenText/listenPicture) 채점은 컨테이너 호출 없음 — 백엔드가 correct만 받아 자체 채점(100/0). `/answer/listen`은 **존재하지 않음**. listenText 선택지는 텍스트형 / listenPicture 선택지는 이미지형 — **유형 고정 (혼합 없음, v1.4)**.

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
| userRT | BigDecimal | X | 유저 평균 RT — 최근 naming 음성 20개 중 발화시간/음절수 최단 10개의 (발화시간 총합 ÷ 음절수 총합). 백엔드 산정. **0개(첫 사용 유저)면 `0` 전송 — v1.4: 구 null 전송 폐지.** 컨테이너는 0 수신 시 이번 녹음 결과를 평균으로 간주 (예외처리 확정) |

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
  "userVoicePath": "shared/1/101/503_user.m4a",
  "articulationRate": 6.67
}
```

| 키 | 타입 | nullable | 비고 |
|----|------|----------|------|
| sessionID / userID | Long | X | ⚠️ `sessionID` — ID 대문자 |
| problemContext | String | X | 정답 확인용 — 원문 구문 |
| userVoicePath | String | X | 공유폴더상 유저 음성 경로 |
| articulationRate | BigDecimal | X | **유저 개인 조음속도 (v1.4 신설)** — 최근 문제풀이(NAMING/SHADOWING/SELF_TALK) 유저 음성 중 SYLLABLES NOT NULL·ARTICULATION_TIME > 0인 것 20개 중 조음속도(`ARTICULATION_TIME÷SYLLABLES`) 최단(=가장 빠른) 10개의 (SYLLABLES 총합 ÷ ARTICULATION_TIME 총합). 백엔드 산정, **소수 2자리**. **0개(첫 사용 유저)면 `0` 전송** — 컨테이너는 0 수신 시 이번 녹음 결과를 평균으로 간주 (예외처리 확정). 명칭은 팀원 전달본 그대로 — `user` 접두 없음 (userRT와 다름) |

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
  "userInfos": { "nickname": "노녹이", "hobbies": "가끔 낚시 갈 정도", "tags": "등산, 골프", "sex": "M", "age": 27, "userMemory": "..." },
  "turnResults": [
    { "turnId": 1, "type": "listenText","context": "사과를 고르세요", "userAnswer": "사과", "score": 100.0 },
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

## 7. 리포트 생성 — 2단계 분리 (v1.2)

> **UX 개선 (2026-09-04 합의):** 리포트를 2단계로 분리 — 문제 8턴 종료 시점에 간이 보고서(AQ+4지표 피드백)를 먼저 확보해 이야기 턴 동안 백그라운드 완성 → 사용자는 종료 후 즉시 간이 보고서를 보고, 상세 보고서(talk/total 피드백)는 완료 후 앱 내 알림으로 수령.

### 7.1 POST /report/problems — 간이 보고서 (문제 8턴 종료 시점)

**트리거:** 유저가 8번째 문제 답안 제출을 마친 시점 — 백엔드가 마지막 채점과 **동시에 자동 호출** (클라 별도 요청 아님). 이야기 턴이 진행되는 동안 컨테이너가 생성을 완료한다. 스텁 지연 2~3초.

#### 요청

```json
{
  "sessionID": 101,
  "userID": 1,
  "turns": [
    { "turnId": 1, "type": "listenText","context": "사과를 고르세요", "userAnswer": "사과", "score": 100.0 },
    { "turnId": 2, "type": "naming",    "context": "포도", "userAnswer": "포도!", "score": 82.0 },
    { "turnId": 3, "type": "shadowing", "context": "나는 오늘 아침에 병원에 갔습니다", "userAnswer": "...", "score": 86.0 },
    { "turnId": 4, "type": "selfTalk",  "context": "감자", "userAnswer": "...", "score": 55.0 }
    // 문제풀이 8턴 전부 (이야기 턴은 이 요청에 포함되지 않음)
  ]
}
```

#### 응답

```json
{
  "sessionID": 101,
  "userID": 1,
  "sessionAQ": 67,
  "sessionFeedbacks": {
    "listenFeedback": "알아듣기 문제를 대부분 정확히 골랐어요.",
    "namingFeedback": "힌트를 사용하면 점수가 내려가요.",
    "shadowingFeedback": "문장을 또박또박 따라했어요.",
    "selfTalkFeedback": "상황 묘사에 핵심 단어가 일부 빠졌어요.",
    "talkFeedback": null,
    "totalFeedback": null
  }
}
```

| 키 | 비고 |
|----|------|
| sessionAQ | **100점 만점 정수** — 8개 문제 점수만으로 산출 (AI 대화 미포함). 소수점 올림은 컨테이너 책임. DB `LEARNING_SESSION.AQ` 즉시 적재 |
| sessionFeedbacks | listen/naming/shadowing/selfTalkFeedback 4종은 non-null. **talkFeedback/totalFeedback은 null** (아직 생성 전 — DB 피드백 6컬럼 중 2컬럼은 이후 §7.2로 채움) |

> **AQ 산정 시점 변경 (v1.2):** 구 설계는 세션 종료 시점에 AQ 포함 전체 리포트 1회. 현재는 `/report/problems`에서 AQ 확정 → 이야기 턴 진행 중 간이 보고서 완성 → 종료 후 상세 피드백만 추가.

### 7.2 POST /report/total — 상세 보고서 (세션 종료/학습 완료 판정 시점)

**트리거:** AI 대화 종료(정상·하드캡) 또는 **학습 완료 판정(유저 4턴째 답변 이후) 중단** 시점 — 백엔드가 1회 호출. **학습 중단(이야기 1~3턴)은 호출하지 않음** (하단 판정 규약). 응답 수신까지 클라는 대기하지 않음 (백그라운드 생성 → 완료 후 앱 내 알림). 스텁 지연 **10초** (실컨테이너 추론 시간 감안).

#### 요청

```json
{
  "sessionID": 101,
  "userID": 1,
  "userMemory": "{기존 누적 userMemory 오파크 텍스트 — 첫 세션이면 null}",
  "turns": [
    { "turnId": 1, "type": "listenText","context": "사과를 고르세요", "userAnswer": "사과", "score": 100.0 },
    { "turnId": 2, "type": "naming",    "context": "포도", "userAnswer": "포도!", "score": 82.0 }
    // 문제풀이 8턴
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
| userMemory | **기존 누적 메모리** (v1.3 신규) — 갱신 기준값. 첫 세션/기존 없으면 `null` |
| turns | 문제풀이 8턴 (이야기 턴은 여기 미포함 — 대화는 talkContext로 별도 전달) |
| talkContext | chatMessage 배열 — 이야기 턴 대화 로그. **학습 완료 판정(4턴째 답변 이후 중단) 시 유저 4턴째 답변까지만 포함** — AI 응답 생성 중이어도 유저 답변까지만 |

#### 응답

```json
{
  "sessionID": 101,
  "userID": 1,
  "userMemory": "{갱신된 userMemory 오파크 텍스트 — 규약 §11 준수}",
  "sessionFeedbacks": {
    "listenFeedback": null,
    "namingFeedback": null,
    "shadowingFeedback": null,
    "selfTalkFeedback": null,
    "talkFeedback": "자연스럽게 대화를 이어갔어요.",
    "totalFeedback": "전반적으로 좋은 흐름이었어요. 특히 듣기가 좋았어요!"
  }
}
```

| 키 | 비고 |
|----|------|
| userMemory | **갱신된 누적 메모리** (v1.3 신규) — 요청의 기존 값 + 이번 talkContext 기반으로 갱신한 결과. DB `USER_PROFILE.USER_MEMORY` 적재. 규약(§10): 갱신할 소득이 없으면 **요청 값과 동일하게 반환**(변경 없음), 실패·누락 시 백엔드는 **기존 값 유지**(데이터 소실 방지). 민감정보 저장 금지(프롬프트 통제, 발표 언급용) |
| sessionFeedbacks | **talkFeedback/totalFeedback만 non-null** — 나머지 4종은 이미 7.1에서 적재됐으므로 null (백엔드는 null 필드 무시하고 2컬럼만 UPDATE) |

> **학습 중단/완료 판정 규약 (v1.6 갱신):** 학습 중단(이야기 1~3턴 중 중단) = `/report/total` **미호출** — 상세 보고서 생성 안 함(간이 보고서는 DB 저장하나 유저 미표시·기록탭 미표시). 학습 완료(유저 4턴째 답변 이후 중단/마치기) = **호출** — talkContext에 유저 4턴째 답변까지만 포함(AI 응답 생성 중이어도 유저 답변까지만). 8턴 하드캡 = 유저 8턴 답변 후 AI 마무리 응답(9번째)까지 포함. STATUS: 중단=`COMPLETED_NO_TALK`, 완료·하드캡=`COMPLETED`.

### 7.3 DB 저장 흐름 (2단계)

```
8턴 종료   → POST /report/problems → AQ + 4지표 피드백 → LEARNING_SESSION UPDATE (AQ, listen/naming/shadowing/selfTalk_feedback)
이야기 중  → 간이 보고서 완성 (클라 조회 가능 상태)
세션 종료  → POST /report/total → talkFeedback + totalFeedback → LEARNING_SESSION UPDATE (부분 UPDATE)
          → LEARNING_SESSION.REPORT_VIEWED_AT는 클라가 상세 보고서 조회한 시점에 기록 (null=미조회 — 알림/버블 판별용)
```

> 스텁 지연: problems 2~3초(이야기 턴 진행 중 완료) / total **10초**(실컨테이너 추론 시간 감안). 알림함·네비 버블은 클라 신규 기능 — 후순위 (데모 제외).

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
| 세션 출제 | `/sessions/today`만 호출 (무작위) | today=무작위 / theme=기획 시나리오 플로우 (엔드포인트 분기 v1.2) |
| LISTEN 세분화 | `listen` 단일 타입 (텍스트 선택지 위주) → **v1.4: listenText/listenPicture로 세분화 — 스텁 미반영, 구현 예정** | listenText=개인화 활용 가능·listenPicture=imageListListening 내 이미지 선택지 (각 1회 포함) |
| LISTEN 선택지 개수 | 고정 → **v1.4: userAQ 등급표(§2) 기반 2~4개 — 스텁 미반영** | 등급표에 따른 개수 (등급표 컨테이너 보유) |
| NAMING 정답 | imageList의 imageName 그대로 → **v1.2: imageListNaming 배열의 imageName** | 컨테이너가 이미지 선택 + 정답 단어 결정 (EASY 이미지 전용 — v1.4) |
| userRT | 과거 기록 없으면 null → **v1.4: 0 전송으로 변경 — 스텁 미반영, 구현 예정** | 첫 사용(0 수신) 시 이번 녹음을 평균으로 간주하는 예외처리 |
| articulationRate | 요청 미전송 → **v1.4 신설 — 스텁 미반영, 구현 예정** | 첫 사용(0 수신) 시 이번 녹음을 평균으로 간주 |
| userText | null (B-2에서 더미 예정) → **✅ B-2 완료: 음성 턴 더미 STT 반환, 첫 호출 null 유지** | 이번 턴 음성의 실제 STT 결과 |
| 점수 | 유사도 시뮬레이션 60~95 + 감점 | 실제 채점 알고리즘 (루브릭 컨테이너 내부) |
| 리포트 | 고정 문구 + AQ 55~85 → **v1.2: 2단계 분리 — problems(AQ+4지표, 2~3s) / total(talk+total, 10s)** | 실제 AQ 산정 + 개인화 피드백 (동일 2단계) |
| 이미지 선택 | 풀에서 무작위 → **v1.2: 타입별 배열(imageListNaming/SelfTalk/Listening) 내에서 무작위** | 유저 정보·맥락 고려 LLM 선택 (해당 배열 내에서) |
| userMemory | **✅ v1.8 갱신 시뮬레이션 구현 완료 (D-4)** — 기존값 있으면 기존+더미 신규 문장 반환, null이면 더미 신규 작성(11a 선언형 스타일). aichat에는 미반환(갱신 지점=/report/total 유일) | 실제 LLM 갱신 (§10 규약) |

## 10. userMemory — 누적 개인화 메모리 규약 (v1.3)

> **개념:** AI 자유대화(STORYTELLING)에서 얻은 유저 개인 맥락을 누적 관리해, 다음 세션의 문제 출제·대화 개시·자유대화에 활용하는 개인화 레이어. **에이전트 메모리 패턴** — 컨테이너(LLM)가 읽고 갱신하고, 백엔드는 저장소로만 동작한다.

### 10.1 라이프사이클

```
[저장] USER_PROFILE.USER_MEMORY CLOB nullable (하드캡 8KB — 백엔드 방어용)
[갱신] 유일한 갱신 지점 = POST /report/total 응답
        요청: 기존 userMemory + turns + talkContext
        응답: 갱신된 userMemory → USER_PROFILE UPDATE (동일 트랜잭션)
        · 갱신할 소득 없음 → 요청 값과 동일하게 반환 (변경 없음)
        · 실패·누락·null → 백엔드는 기존 값 유지 (데이터 소실 없음, 다음 세션에서 재시도)
        · 첫 세션은 기존 값 null → 대화 내용만으로 신규 작성
[활용] POST /sessions/today·theme (출제 개인화) + POST /aichat (대화 개인화)
        → userInfos.userMemory로 주입. 매 턴 주입 비용은 무시 가능 수준
[파기] 회원탈퇴 — USER_PROFILE 삭제에 자동 포함 (B-1 FK 역순 삭제 흐름)
```

### 10.2 백엔드 책임 경계

| 항목 | 백엔드 | 컨테이너(LLM 팀) |
|------|--------|------------------|
| 형식(스키마) | **비관여** — 오파크 텍스트 통째로 전달/저장 | 결정·문서화 (예: JSON 항목 배열) |
| 갱신 로직 | — | /report/total에서 LLM 갱신 |
| 길이 관리 | 하드캡 8KB만 (초과 시 절단 저장 — 방어용) | 항목 수·길이를 프롬프트로 관리 (예산 내 유지) |
| 민감정보 | — | 프롬프트 레벨 통제 (§10.3) — 기술적 제약은 별도 검토 과제 |
| 파기 | 회원탈퇴 시 행 삭제 | — |

### 10.3 내용 규약 (컨테이너 프롬프트에 주입할 지침 — LLM 팀 가이드라인)

| 항목 | 규약 |
|------|------|
| 저장 대상 | 개인화에 쓸 유저 맥락만 — 가족·취미·말버릇·선호·대화 스타일·지난 세션에서 말한 사건 |
| 저장 금지 | **민감정보 (의료 정보, 질병, 수술, 주소, 연락처 등)** — 발표 시 "민감정보는 저장하지 않도록 노력했다" 언급용 통제. 프롬프트 레벨 제한 |
| 관리 방식 | 선언형 짧은 문장 (에이전트 메모리 스타일). 항목 수 상한 (예: 최대 10항목) — 초과 시 오래되거나 중복·저가치 항목 통합/교체 |
| 갱신 원칙 | 대화에서 새 사실만 추가, 기존 항목이 틀렸으면 교체. 매번 전체 재작성하지 않음 |
| 불확실 처리 | 확실하지 않은 추론은 저장하지 않음 — 대화에 직접 나온 것만 |

> 본 문서의 내용 규약은 **가이드라인 예시** — 최종 규칙은 LLM 팀이 컨테이너 내부 프롬프트로 구현. 프롬프트 예시는 03a 별첨(`11a_memory_prompt_example.md`) 참조.

## 11. 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-09-04 | 초안 — demo 브랜치 실구현 DTO(`AiContainerDtos.kt`) 역추적. 엔드포인트별 전체 JSON 예시 포함. 스텁/실컨테이너 차이 표 (§9) |
| v1.1 | 2026-09-03 | B-2/B-3 반영 — §2 요청에 namingImageIds/selfTalkImageIds 선택 필드 추가(백엔드 조건 필터+완화 규약), §6 스텁 userText 더미 STT 수정 완료 표기, §9 차이표 갱신 |
| v1.2 | 2026-09-04 | **컨테이너 협의 반영 (1):** §2 엔드포인트 분기 — `/sessions/today`(테마 랜덤+무작위 출제) / `/sessions/theme`(기획 시나리오 플로우), 요청·응답 필드 공통. **imageList 3분할** — imageListListening/Naming/SelfTalk (분류: TAG_PATH 있음=SELF_TALK, TAG 없음+CUE 있음=NAMING(+LISTEN 공용), 둘 다 없음=LISTEN). 구 namingImageIds/selfTalkImageIds 폐지. §7 리포트 2단계 분리 — `/report/problems`(8턴 종료, AQ+4지표, 2~3s) / `/report/total`(종료, talk+total, 10s) + 조기종료 COMPLETED_NO_TALK 규약 + §7.3 DB 저장 흐름 |



| v1.7 | 2026-09-04 | **컨테이너 협의 확정 (7) — 대표점수 테이블 이동:** §2 userAQ 출처 갱신 — USER_PROFILE → USER_REPRESENTATIVE_SCORES (대표점수 5종 통합). 컨테이너 입장 변화 없음 |
| v1.3 | 2026-09-04 | **컨테이너 협의 반영 (2) — userMemory 개인화:** §1.1 userInfos 개편 — `userMemory` 신설, `likes`→`hobbies` rename, `tags` 신설(선택 태그 최대 5개, 쉼표 문자열), age=BIRTH_DATE 기반 산정. §7.2 /report/total — 요청에 기존 `userMemory` 추가, 응답에 **갱신된 `userMemory`** 추가(변경 없으면 동일값 반환, 실패 시 기존 유지). **§10 신설** — userMemory 라이프사이클/책임 경계/내용 규약(민감정보 저장 금지 — 프롬프트 통제). 백엔드는 오파크 CLOB 저장소(USER_PROFILE.USER_MEMORY, 하드캡 8KB) 
| v1.8 | 2026-09-05 | **D-4 구현 완료 반영:** §9 차이표 userMemory 행 갱신 — 스텁 갱신 시뮬레이션 구현 완료 표기(기존+더미 신규/신규 작성, aichat 미반환). 백엔드 실측: ReportRequest/Response userMemory 필드 동작, 소실 방지(null 응답→기존 유지)·하드캡 8192문자 절단 실측 완료. 리포트 2단계 분리(/report/problems·total 엔드포인트 분리)는 D-5 — 현재는 동기 finish 1회 호출에 §7.2 total 규약만 부착 ||