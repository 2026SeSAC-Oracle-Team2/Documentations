# 클라이언트 ↔ 백엔드 API 명세서 (Android ↔ Spring Boot)

> **버전:** v1.0 (2026-09-04) — **실제 구현 코드 기준** (demo 브랜치, VM ~/app 44fe801+9b32634, Android 514fdce)
> **작성 방식:** 구현된 컨트롤러/DTO를 역추적해 작성 — 스펙 문서(`05_API_Design.md`)와의 차이는 ⚠️ 표기
> **Base URL:** `http://{VM주소}` (:80, nginx 경유) · 응답 봉투: `{ success, data, timestamp }` / 에러: `{ success: false, error: { code, message, detail, timestamp } }`

## 0. 인증 현황 (중요)

| 구간 | 상태 |
|------|------|
| Auth/User API (`/auth/**`, `/users/**`) | **JWT 필수** (`anyRequest().authenticated()`) |
| 세션/음성/콘텐츠 (`/sessions/**`, `/voice/**`, `/content/**`) | ⚠️ **permitAll (dev 임시)** — userId를 쿼리파라미터로 전달하는 임시 계약. 운영 전 JWT 인증 전환 필수 |

> 만료 응답은 **실측 403** (스펙상 401과 다름) — 클라 TokenAuthenticator는 401+403 모두 처리.

## 1. 인증 API — `/api/v1/auth`

### 1.1 POST /api/v1/auth/firebase — 로그인
```json
// 요청
{ "id_token": "<Firebase ID Token>" }
// 응답 data
{
  "access_token": "...", "refresh_token": "...", "expires_in": 900,
  "user": { "id": 1, "uuid": "...", "email": "a@b.c", "nickname": null,
            "profile_image_url": null, "level": 1, "created_at": "..." },
  "is_new_user": true
}
```
- ⚠️ 응답 키는 snake_case (`access_token` 등) — 클라 TokenManager 저장 키와 일치

### 1.2 POST /api/v1/auth/refresh — 토큰 갱신
```json
// 요청 — ✅ camelCase (v1.98 수정: 구 snake_case는 Jackson 파싱 500)
{ "refresh_token": "..." }   // 헤더 X-Refresh-Token도 지원
// 응답 data
{ "access_token": "...", "expires_in": 900 }
```

### 1.3 POST /api/v1/auth/logout — 로그아웃 (Bearer)

## 2. 사용자 API — `/api/v1/users`

| Method | Path | 요청 | 응답 data | 비고 |
|--------|------|------|-----------|------|
| GET | `/me` | — | UserDto (id/uuid/email/nickname/profile_image_url/level/created_at) | 만료 시 403 → 클라 무음 refresh |
| PATCH | `/me` | `{ "nickname": "..." }` | UserDto | 닉네임 수정 |
| DELETE | `/me` | — | **204 No Content** | ✅ **B-1 수정 완료** — FK 역순 하드딜리트(TURN_IMAGE→VOICE_RECORD→TURN→LEARNING_SESSION→USER_PROFILE→APP_USER) + OCI 유저 파일(userfiles {userUUID}/ 하위) 커밋 후 삭제(실패 시 로그만) + Firebase afterCommit 삭제 |
| POST | `/me/profile-image` | multipart `image` | `{ "profile_image_url": "..." }` | 5MB |
| GET | `/me/profile-image` | — | 이미지 바이너리 (프록시 스트리밍) | 캐시버스터 `?v=` 권장 |

## 3. 세션 플로우 API — `/api/v1/sessions` (스텁 AI 컨테이너)

> 구현: `SessionFlowController` + `SessionFlowService`. 모든 엔드포인트에서 **`userId`는 쿼리파라미터** (dev 임시 계약 — ⚠️ 스펙 문서에는 Bearer 기반으로 기술되어 있음)

### 3.1 POST /api/v1/sessions/v2 — 세션 생성 ("오늘의 학습")

| 파라미터 | 타입 | 위치 |
|----------|------|------|
| `userId` | Long | query |

**동작:** 테마 랜덤(`demo.themes` 프로퍼티, 현재 TEST) → IMAGE_THEMA 이미지 풀 → StubClient 8문제 생성 (2~3초) → TURN 8행 INSERT(PENDING) + VOICE_RECORD AI 행 → 응답

```json
// 응답 data — SessionCreateData
{
  "sessionId": 101,
  "theme": "TEST",
  "turns": [
    {
      "turnId": 501, "turnNumber": 1, "type": "LISTEN",
      "ttsUrl": "/api/v1/voice/9001",       // 상대경로 — Base URL 결합
      "passage": "여기서 금요일이 며칠인지 말해보세요",
      "choices": [ { "order": 1, "mediaType": "TEXT", "context": "8월 12일" },
                    { "order": 2, "mediaType": "TEXT", "context": "8월 20일" } ],
      "imageId": null, "imageUrl": null, "hintAvailable": null
    },
    {
      "turnId": 502, "turnNumber": 2, "type": "NAMING",
      "ttsUrl": "/api/v1/voice/9002",
      "imageId": 33, "imageUrl": "/api/v1/content/images/33/file",
      "hintAvailable": 2
    }
    // SHADOWING: passage + ttsUrl / SELF_TALK: imageId + imageUrl
  ]
}
```
- ⚠️ `mediaType`은 실측 **소문자** `text` | `image` (스펙 문서 대문자 TEXT|IMAGE와 다름)
- ⚠️ 경로가 실구현 `/v2` — 스펙 문서와 차이. 클라는 v2 사용 중
- type은 대문자 `LISTEN|NAMING|SHADOWING|SELF_TALK` (클라 매핑 확인됨)

### 3.2 답안 제출 (타입별)

| 타입 | Path | 요청 | 응답 data |
|------|------|------|-----------|
| LISTEN | `POST /{sessionId}/turns/{turnId}/listen` | JSON `{ "selected": 2 }` — **1-based order (Int)** | `ListenSubmitData` |
| NAMING | `POST /{sessionId}/turns/{turnId}/naming` | multipart: `userId`(query) + `file`(m4a) | `VoiceSubmitData` |
| SHADOWING | `POST /{sessionId}/turns/{turnId}/shadowing` | 동일 | `VoiceSubmitData` |
| SELF_TALK | `POST /{sessionId}/turns/{turnId}/selftalk` | 동일 | `VoiceSubmitData` |

```json
// ListenSubmitData — LISTEN은 백엔드 자체채점, 즉시 응답
{ "turnId": 501, "score": 100, "correct": true }

// VoiceSubmitData — 스텁 채점 0.8~1.5초 후
{
  "turnId": 502,
  "score": 82.0,
  "voiceRecordId": 9002,
  "userVoiceEval": {
    "durationSecond": 7, "syllables": 12,
    "speakingTime": 2.4, "articulationTime": 1.8,
    "text": "포도!"          // STT — TURN.answer_text 적재
  }
}
```
- 제출 시 백엔드: OCI 업로드 + VOICE_RECORD USER 행(발화지표 3종) + TURN.score/answer_text + status=SCORED
- 클라는 응답 score를 **누적** (세션상세 GET API 부재 — 결과 화면 Intent 전달 방식)

### 3.3 POST /{sessionId}/turns/{turnId}/hint — 힌트 요청

```json
// 응답 data — HintData (의미단서→조음단서 고정 순서)
{ "hintOrder": 1, "cueType": "SEMANTIC", "text": "퍼런색에 동그란 열매가 많은 과일" }
// 2번째: { "hintOrder": 2, "cueType": "ARTICULATORY", "text": "ㅍㄷ" }
// 3번째 요청: 에러 E0401 (힌트 소진) — 클라는 사전 비활성 처리됨
```
- 내부: IMAGE_RESOURCE.SEMANTIC_CUE/ARTICULATORY_CUE 조회, TURN.hints_shown 증가

### 3.4 POST /{sessionId}/turns/talk — 이야기 턴 (STORYTELLING)

| 파라미터 | 위치 | 비고 |
|----------|------|------|
| `userId` | query | |
| `file` | multipart, **required=false** | **첫 호출은 multipart 없이 일반 POST** (AI가 먼저 개시). 이후 턴은 음성 첨부 |

```json
// 응답 data — TalkData (스텁 1~2초)
{ "turnId": 511, "turnNumber": 9,
  "aiText": "아까 문제 푸느라 고생했네요! 오늘 하루 어땐어요?",  // TURN.prompt_text
  "userText": "오늘은 카페에 갔어요" }   // 이번 턴 유저 STT — ✅ B-2 수정 완료: 음성 턴은 스텁 더미 STT 반환(첫 호출은 null 유지), 클라는 null 시 "(인식된 말 없음)" 표시
```
- 데모 하드캡: `demo.talk-turn-limit=3` — 4번째 제출 시 E0401(세션 이야기 턴 소진) → 클라 종료 안내
- 조기종료: 클라가 그냥 `/finish` 호출하면 됨

### 3.5 POST /{sessionId}/finish — 세션 종료 + 리포트

```json
// 응답 data — FinishData (스텁 2~3초, 동기 — 클라 로딩 대기)
{
  "sessionAQ": 67,
  "feedbacks": {
    "listenFeedback": "...", "namingFeedback": "...", "shadowingFeedback": "...",
    "selfTalkFeedback": "...", "talkFeedback": "...", "totalFeedback": "..."
  }
}
```
- 백엔드: StubClient /report → LEARNING_SESSION에 AQ+피드백 6종 저장, status=COMPLETED
- userAQ 산정식: 최근 20세션 AQ 상위 10 평균 (0개 null) — /sessions/v2에 사용
- ⚠️ **턴별 점수는 응답에 없음** — 클라가 제출 응답 score를 누적해 결과 화면에 전달 (세션상세 GET API는 미구현, P5 과제)

## 4. 음성/콘텐츠 스트리밍

| Method | Path | 응답 | 비고 |
|--------|------|------|------|
| GET | `/api/v1/voice/{voiceRecordId}` | `audio/mpeg` (60s 캐시) | **스텁**: voiceRecordId % 4로 tts_samples mp3 반환. 실모드: DB voice_file_path → OCI 스트림 |
| GET | `/api/v1/content/images/{imageId}/file` | image/* (300s 캐시) | OCI problemfiles 버킷에서 스트림 (콘텐츠 이미지 전용 프록시 — 9b32634) |
| POST | `/api/v1/voice/upload` | 업로드 (P3-19, 녹음 테스트용) | OCI userfiles 적재 |

## 5. 구현↔스펙 차이 요약 (P3-29 실계약 정리)

| 항목 | 스펙(05 문서) | 실구현 |
|------|---------------|--------|
| 세션 생성 경로 | `POST /sessions` | `POST /sessions/v2` |
| userId 전달 | Bearer JWT | **쿼리파라미터** (permitAll 임시) |
| LISTEN selected | choice ref (문자열) | Int (1-based order) |
| mediaType | `TEXT\|IMAGE` | `text\|image` 소문자 |
| 만료 응답 | 401 | **403** |
| refresh 요청 | snake_case | **camelCase** (구값은 500 버그였음) |
| 톡 첫 호출 | — | multipart 없이 일반 POST |
| TURN 유형 코드 | 대문자 유지 | 대문자 유지 (API만 camelCase 검토) |

## 6. 스텁(AI 컨테이너) ↔ 실컨테이너 전환 — 어디를 어떻게 고치나

> 결론: **백엔드 코드 수정 0건.** 프로퍼티 1줄 + 컨테이너 배포 + (권장) base-url 지정만.

### 6.1 전환 포인트 (단 하나)

```
~/app/src/main/resources/application.yml

ai:
  container:
    mode: stub     ← real 로 변경
    base-url: http://ai-container:8000   ← (주입 TODO 상태 — 아래 6.3)

demo:
  themes: TEST            ← 운영: TEST,HOSPITAL,CAFE
  talk-turn-limit: 3      ← 운영: 8 (또는 제거로 무제한)
```

- 선택 로직: `@ConditionalOnProperty(name="ai.container.mode", havingValue="stub"|"real")` — 인터페이스 `AiContainerClient` 구현체 2종이 스위칭됨
- 스텁의 자연 지연(2~3s/0.8~1.5s/1~2s/2~3s)은 StubClient 내부 Thread.sleep이라 real 전환 시 자동 소멸 — 클라 로딩 UI는 실지연에 그대로 대응

### 6.2 AI 컨테이너(FastAPI) 측이 맞춰야 할 것 — 이미 고정돼 있음

계약 = `03_AI_Container_Contract.md`. 실구현 DTO(`dto/aicontainer/AiContainerDtos.kt`)가 그 계약을 그대로 구현 — **컨테이너는 아래 JSON 키를 정확히 지켜야 함:**

| 엔드포인트 | 요청 (BE→컨테이너) | 응답 (컨테이너→BE) | 주의 키 |
|-----------|---------------------|---------------------|---------|
| POST /sessions | `{sessionId, thema, imageList:[{imageId,imageName}], userID, userInfos{nickname,likes,sex,age}, userAQ}` | `{sessionId, userID, problemList:[{turnId(로컬1~8), type(소문자), ttsPath, passage, perType}]}` | userID/userID (ID 대문자), type 소문자, perType.listen: correct+options, naming: correct, selfTalk: image |
| POST /answer/naming | `{sessionID, userID, problemContext, userVoicePath, hintCount, userRT}` | `{sessionID, userID, scoreNaming, userVoiceEval}` | score* 키 이름 타입별 상이 |
| POST /answer/shadowing | `{sessionID, userID, problemContext, userVoicePath}` | `{sessionID, userID, scoreShadowing, userVoiceEval}` | |
| POST /answer/selfTalk | `{sessionID, userID, problemImage(이름), problemTag(tags.json 원문), userVoicePath}` | `{sessionID, userID, scoreSelfTalk, userVoiceEval}` | |
| POST /aichat | `{sessionID, userID, userInfos, turnResults[], context[{speaker,text}], userVoicePath?}` | `{sessionID, userID, llmResponse, userText}` | 첫 호출 context 빈 배열 + userVoicePath 없음 |
| POST /report | `{sessionID, userID, turns[], talkContext[]}` | `{sessionID, userID, sessionAQ, sessionFeedbacks{6종}}` | |

- userVoiceEval 공통: `{durationSecond, syllables, speakingTime, articulationTime, text}` — 발화지표 3종 rename 반영
- **ttsPath**: 공유폴더 실제 경로 (`{userUUID}/{sessionID}/{로컬turnId}_ai.mp3`). real 전환 시 docker-compose에 볼륨 마운트 추가 필수 (BE와 컨테이너가 같은 경로 공유)

### 6.3 real 전환 시 체크리스트 (배포 시 수행)

| # | 작업 | 위치 |
|---|------|------|
| 1 | `ai.container.mode: real` | application.yml (VM ~/app) |
| 2 | `ai.container.base-url` 주입 활성화 | `RealAiContainerClient.kt` — 현재 하드코딩 `http://localhost:8000` + TODO 주석. 같은 Docker 네트워크면 서비스명(ai-container:8000), 호스트 실행이면 localhost:8000. **이 줄만 코드 수정 필요** |
| 3 | compose에 AI 컨테이너 서비스 + 공유폴더 볼륨 (BE↔컨테이너 동일 마운트) | SeSAC_SpeechApp_Deployment |
| 4 | `demo.*` 프로퍼티 운영값 | themes 3종, talk-turn-limit 8 |
| 5 | SecurityConfig permitAll 회수 (sessions/voice/content → JWT) | SecurityConfig.kt — 운영 전 필수 |
| 6 | 스텁 스트리밍 → OCI 스트리밍 교체 | VoiceStreamController (주석에 TODO 있음) |

### 6.4 전환 후 바뀌는 것 / 안 바뀌는 것

| 구분 | 내용 |
|------|------|
| 그대로 (코드 무수정) | Android 앱 전체 / 클라↔BE API / DB 스키마 / TURN INSERT·리매핑 로직 / 힌트 / 리포트 저장 |
| 바뀌는 응답 성격 | 지연시간이 스텁 시뮬레이션 → 실제 추론 시간 / userText가 실제 STT 결과로 수신 (스텁은 null — B-2) / TTS 실제 음성 |
| 리스크 포인트 | (1) base-url 하드코딩 1곳 (6.3-2) (2) 공유폴더 마운트 경로 일치 (3) 컨테이너 응답 JSON 키 정확성 — 위 표가 계약의 전부 (4) 타임아웃 규약 미정 — 컨테이너 응답 지연 시 기본 타임아웃에 걸릴 수 있음 (고도화 과제) |

## 7. 알려진 이슈 (백엔드 후속 세션 B-1~B-3)

> ✅ **2026-09-03 전부 수정 완료** (demo 브랜치) — 검증 실측은 아래 상태 표.

| 이슈 | 상태 |
|------|------|
| 회원탈퇴 500 (FK 위발) | ✅ B-1 해결 — FK 역순 하드딜리트 + OCI 파일 정리. 실측: 학습 데이터 보유 계정 탈퇴 204 → DB 잔존 0 → 재가입 정상 |
| 스텁 aichat userText null | ✅ B-2 해결 — 음성 턴 더미 STT 반환(첫 호출 null 유지, 03a §6 규약). 실측: TURN.answer_text DB 적재 확인 |
| 문제 출제 이미지 풀 필터 (NAMING=cue 필수, SELF_TALK=tag 필수) | ✅ B-3 해결 — imageList 구성 시점 필터 + 스텁 조건 선택 보정. 실측: NAMING cue 적합율 100% (SELF_TALK은 tag 데이터 1개뿐 → 완화 로그와 함께 전체 풀 폴백 — tag 데이터 보충 시 자동 적용) |

## 8. 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-09-04 | 초안 — demo 브랜치 실구현 역추적 작성 (SessionFlowController/SessionFlowDtos/AiContainerClient/SecurityConfig/application.yml 실측) |
| v1.1 | 2026-09-03 | B-1~B-3 수정 반영 — DELETE /me 204 확정(B-1 FK 역순 하드딜리트+OCI 정리), talk userText 스텁 더미 STT(B-2), §7 이슈 전건 해결 표기 |