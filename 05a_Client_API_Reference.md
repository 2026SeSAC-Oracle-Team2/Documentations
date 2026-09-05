# 클라이언트 ↔ 백엔드 API 명세서 (Android ↔ Spring Boot)

> **버전:** v1.6 (2026-09-06) — **실제 구현 코드 기준** (demo 브랜치, VM ~/app 9521f2b D-5, Android 514fdce) + **컨테이너 협의 확정 (3·6·7) 반영** (구현 예정분은 ⏳ 표기)
> **작성 방식:** 구현된 컨트롤러/DTO를 역추적해 작성 — 스펙 문서(`05_API_Design.md`)와의 차이는 ⚠️ 표기
> **Base URL:** `http://{VM주소}` (:80, nginx 경유) · 응답 봉투: `{ success, data, timestamp }` / 에러: `{ success: false, error: { code, message, detail, timestamp } }`

## 0. 인증 현황 (중요)

| 구간 | 상태 |
|------|------|
| Auth/User API (`/auth/**`, `/users/**`) | **JWT 필수** (`anyRequest().authenticated()`) — **`GET /users/me/sessions/history` 포함 (v1.6 D-5 신설, JWT 경로)** |
| 세션/음성/콘텐츠 (`/sessions/**`, `/voice/**`, `/content/**`) | ⚠️ **permitAll (dev 임시)** — userId를 쿼리파라미터로 전달하는 임시 계약. 운영 전 JWT 인증 전환 필수 (§6.3 잔여) |
| 세부 보고서 `GET /sessions/{id}/report` (v1.6 D-5) | permitAll 경로지만 **userId 쿼리파라미터 소유 검증** — 세션 소유 유저만 조회 허용(타 유저 E0400), 중단 세션 E0404. 클라는 기존 세션 플로우 임시 계약과 동일 패턴(userId 쿼리파라미터 병용) |

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
| GET | `/me` | — | UserDto (id/uuid/email/nickname/profileImageUrl/level/createdAt + **hobbies/sex/birthDate/tags/userAq** — D-3 확장) | 만료 시 403 → 클라 무음 refresh. userAq null = 설문 미응답(재노출 판별 기준) |
| PATCH | `/me` | `{ "nickname": "...", "hobbies": "...", "sex": "M", "birthDate": "1990-05-12", "tagIds": [1,2,9] }` (전부 nullable — 부분 업데이트) | UserDto | **D-3 확장** — 닉네임·취미·성별·생년월일(ISO yyyy-MM-dd 고정, 파싱 실패 E0400)·태그. tagIds는 **전량 교체**(기존 DELETE 후 INSERT — 클라가 항상 현재 선택 전체 전송, null=변경 없음, 명시적 `[]`=전체 삭제). >5개 → E0400 / 없는 tag_id → E0404 |
| GET | `/me/tags` | — | `{ "tags": [ { "tagId": 1, "tag": "건강관리" }, ... 15종 ] }` | **D-3 신설** — 태그 마스터 15종 (가입 화면 버블용). order by tagId |
| POST | `/me/survey` | `{ "answers": [1~5 정수 5개] }` (문항 순서대로, 길이 5 고정) | `{ "userAq": 30, "user": UserDto }` | **D-3 신설** — 가입 설문 접수 (06 §5.2). **산출 주체 = 서버**(요청은 answers 원문만). 총점=Σ(answer×4) → 20~61→30 / 62~80→70 / 81~100→90. USER_REPRESENTATIVE_SCORES upsert(행 없으면 INSERT, 있으면 갱신 — 재노출 케이스, 중복 응답 허용·405 거부 금지). 응답 원문 미저장(환산 AQ만). 문항 텍스트는 앱 고정 — 서버 저장 없음 |
| GET | `/me/scores` | — | `{ "userAq": 73, "listen": 75, "naming": 74.75, "shadowing": 79.75, "selfTalk": 79.75 }` | **D-3 신설 (§8.1 실구현)** — 대표점수 조회. REP_SCORES 단일 SELECT — null은 null 전달(클라 폴백: 방사형 0 표시+안내문) |
| DELETE | `/me` | — | **204 No Content** | ✅ **B-1 수정 완료** — FK 역순 하드딜리트(**D-3 확장: TURN_IMAGE→VOICE_RECORD→TURN→LEARNING_SESSION→USER_PROFILE_TAGS→USER_REPRESENTATIVE_SCORES→USER_PROFILE→APP_USER 8단계**, TAGS 마스터는 보존) + OCI 유저 파일(userfiles {userUUID}/ 하위) 커밋 후 삭제(실패 시 로그만) + Firebase afterCommit 삭제 |
| POST | `/me/profile-image` | multipart `image` | `{ "profile_image_url": "..." }` | 5MB |
| GET | `/me/profile-image` | — | 이미지 바이너리 (프록시 스트리밍) | 캐시버스터 `?v=` 권장 |

> **UserDto 확장 (D-3, 하위호환):** 기존 필드 유지 + `hobbies`(취미 자유 텍스트)/`sex`/`birthDate`(ISO yyyy-MM-dd 문자열)/`tags`(선택 태그 쉼표 문자열 — 03a §1.1 형식 "등산, 골프")/`userAq`(REP_SCORES 조회, null 허용) 추가. 제거된 필드 없음 — 기존 클라 파싱 무영향. 오류 코드: IllegalArgumentException→E0400 / NoSuchElementException→E0404 / IllegalStateException→E0401 (GlobalExceptionHandler 매핑).

## 3. 세션 플로우 API — `/api/v1/sessions` (스텁 AI 컨테이너)

> 구현: `SessionFlowController` + `SessionFlowService`. 모든 엔드포인트에서 **`userId`는 쿼리파라미터** (dev 임시 계약 — ⚠️ 스펙 문서에는 Bearer 기반으로 기술되어 있음)

### 3.1 세션 생성 — POST /api/v1/sessions/today · /sessions/theme (v1.6 D-5 2종 분기)

| 파라미터 | 타입 | 위치 | 비고 |
|----------|------|------|------|
| `userId` | Long | query | 공통 |
| `thema` | String | query | **/theme 전용** — TEST/HOSPITAL/CAFE (이외 E0400 "허용되지 않는 테마"). 대소문자 무관(소문자 cafe→CAFE) |

**동작 (v1.6 D-5):** 엔드포인트 2종 분기 —
- `POST /sessions/today?userId=26` — 테마 **랜덤**(`demo.themes` 프로퍼티, 현재 TEST) + 무작위 출제 → LEARNING_SESSION.type=`today`
- `POST /sessions/theme?userId=26&thema=TEST` — 테마 **고정** → LEARNING_SESSION.type=`theme` (기획 시나리오 플로우는 컨텐츠 확정 후 — 현재 스텁은 today와 동일 무작위 출제, 컨테이너 엔드포인트만 분기)
- 공통: IMAGE_THEMA 이미지 풀 → 컨테이너 세션 생성 (스텁 2~3초) → TURN 8행 INSERT(PENDING) + VOICE_RECORD AI 행 → 응답
- `POST /sessions/v2?userId=26` — **하위호환 유지**(클라 데모가 사용 중) — today와 동일 동작

```json
// 응답 data — SessionCreateData (v1.6: type 필드 추가)
{
  "sessionId": 101,
  "theme": "TEST",
  "type": "today",      // D-5: today | theme — 요청 엔드포인트에 따른 세션 종류
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
- type은 대문자 `LISTEN_TEXT|LISTEN_PICTURE|NAMING|SHADOWING|SELF_TALK` (✅ v1.2 협의 세분화 **D-2 실구현 완료** — 03a v1.4 계약. 선택지는 LISTEN_TEXT=텍스트형만/LISTEN_PICTURE=이미지형만)

### 3.1a 클라 세션 흐름 규약 (v1.3 협의 — 구현 예정)

| 항목 | 규약 |
|------|------|
| 제출 제한 | **30초 통일** — 모든 문제 + AI 대화 답변. 30초 도달 시 클라가 녹음 컷 → multipart 강제 제출 |
| 대기 카운트다운 | LISTEN·SHADOWING 3초 후 TTS 재생 / NAMING·SELF_TALK 5초 (사진 관찰) |
| SHADOWING 흐름 | TTS 재생 종료 → 3초 후 녹음 시작 → 30초. **[다시 듣기] 없음** (마이크 오염 방지) |
| LISTEN 30초 도달 | 선택 누름=최근 선택지로 제출 / 미선택=오답 처리 제출. [다시 듣기] 제공(카운트다운 진행 중에도 무관) |
| NAMING 힌트 | 30초 카운트다운과 무관 (시간 정지 없음) |
| 안내 | 텍스트만 — 안내 TTS 없음. 마이크 아이콘 대신 "녹음 시작/완료" 굵은 텍스트 (시니어 UI) |
| 버튼 분리 | "제출"(시간 제한)과 "다음으로"(제한 없음 — 이동 자유) 구분 |
| AI 대화 | [음성으로 답변하기] 직접 클릭 시작, 녹음 30초 제한 표시, [녹음 완료]/30초 도달 → 자동 제출. LLM 로딩 중 "덕분이가 답변을 생각중이에요" |
| 중단/완료 | 1~3턴 중단=우는 덕분이 팝업(학습 중단 — 상세 보고서 없음) / 4턴째 답변 후 [학습 마치기] 전환(학습 완료 — total 호출) / 8턴 하드캡=AI 마무리 응답 후 [학습 결과 보기] |

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
- 조기종료(구 데모): 클라가 `/finish` 호출 — ⏳ **v1.6 협의로 개편 예정**: 1~3턴 중단=학습 중단 판정(우는 덕분이 팝업 → total 미호출) / 4턴째 답변 후 [학습 마치기]=학습 완료 판정(total 호출, 유저 4턴 답변까지만). 데모 구현은 구 규약 유지 — 백엔드 작업 세션에서 판정 로직 반영

### 3.5 POST /{sessionId}/finish — 세션 종료 + 간이 보고서 (v1.6 D-5 재편)

```json
// 응답 data — FinishData (동기 즉시 반환 — 상세 보고서는 백그라운드, 대기 없음)
{
  "sessionAQ": 81,
  "feedbacks": {
    "listenFeedback": "...", "namingFeedback": "...", "shadowingFeedback": "...",
    "selfTalkFeedback": "...", "talkFeedback": null, "totalFeedback": null
  }
}
```
- **v1.6 2단계 재편 (03a §7):** AQ+4지표 피드백은 8번째 문제 채점 완료 시점에 백엔드가 자동 확보(/report/problems 백그라운드) — finish는 그 **간이 보고서를 응답**으로 돌려준다. talk/total 피드백은 **항상 null** — 세부 보고서(§8.3)에서 수령 (클라 D-6이 이 계약을 따라감)
- **중단/완료 판정 (03 계약서 §9.3):** 유저 이야기 답변(=STORYTELLING answer_text 행 수) 기준 — **1~3턴 = 학습 중단** → STATUS=`COMPLETED_NO_TALK` + 상세 보고서 생성 안 함(talk/total NULL 유지·기록탭 미표시) / **4턴 이상 = 학습 완료** → STATUS=`COMPLETED` + /report/total 백그라운드 호출(스텁 10초 — talk/total 피드백 + userMemory 갱신 완료 후 세부 보고서 조회 가능) / **8턴 하드캡** = 유저 8턴째 답변 후 AI 마무리 응답까지 생성 가능
- STATUS: COMPLETED / COMPLETED_NO_TALK / IN_PROGRESS — 이력 필터(§8.2)는 `COMPLETED_NO_TALK 제외 + AQ NOT NULL`
- userAQ 출처 (v1.6 교체): **REP_SCORES.USER_AQ 캐시 직접 조회** — 산정식 실행 없음 (03 계약서 §10 v1.7 계약)
- ⚠️ **턴별 점수는 응답에 없음** — 클라가 제출 응답 score를 누적해 결과 화면에 전달 (세부 보고서 §8.3에서 턴별 답변도 수령 가능)

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
| POST /report/problems (✅ v1.6 2단계 분리) | `{sessionID, userID, turns[8문제]}` | `{sessionID, userID, sessionAQ, sessionFeedbacks(4지표 non-null·talk/total null)}` | 백엔드가 8번째 채점 완료 감지 후 자동 호출(백그라운드) |
| POST /report/total (✅ v1.6 2단계 분리) | `{sessionID, userID, userMemory, turns[8문제], talkContext[유저 4턴까지만]}` | `{sessionID, userID, userMemory(갱신값), sessionFeedbacks(talk/total non-null·4지표 null)}` | finish에서 유저 talk 답변 4턴 이상 시 백그라운드 호출 — 중단(1~3턴) 미호출 |

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

## 8. 대시보드 / 세부 보고서 API (v1.4 협의 — 8.1은 D-3 실구현 완료, 8.2/8.3은 ⏳ D-5)

> 대시보드 탭 실구현 + 세부 보고서 화면용. 인증: JWT (Bearer). 스키마: USER_REPRESENTATIVE_SCORES + LEARNING_SESSION(SESSION_NAME 포함, 04 v2.6).

### 8.1 GET /api/v1/users/me/scores — 대표점수 (대시보드 방사형) — ✅ **D-3 실구현 완료 (2026-09-05)**

```json
// 응답 data
{
  "userAq": 55,
  "listen": 78.5, "naming": 70.0, "shadowing": 82.4, "selfTalk": 65.2
}
```

- 출처: USER_REPRESENTATIVE_SCORES (USER_AQ + 4지표 — LISTEN은 LISTEN_TEXT/LISTEN_PICTURE 통합)
- null = 캐시 미산출(설문 미응답 or 실세션 0) — 클라는 방사형에서 0 표시 후 "학습을 시작해보세요" 등 폴백

### 8.2 GET /api/v1/users/me/sessions/history — 지난 학습 카드 리스트 — ✅ **D-5 실구현 완료 (2026-09-06)**

```json
// 응답 data — 실측 (user 26)
{
  "sessions": [
    { "sessionId": 76, "sessionName": "오늘의 학습 - TEST", "createdAt": "2026-09-06T01:50:09.527617", "aq": 81 },
    { "sessionId": 68, "sessionName": "오늘의 학습 - TEST", "createdAt": "2026-09-06T00:43:10.185821", "aq": 63 }
  ]
}
```

- 인증: **JWT 필수** (§0 — userUuid 기반 소유 스코프)
- 조회 조건 (v1.6 규약 확정): **STATUS != COMPLETED_NO_TALK AND AQ IS NOT NULL** — AQ null = 간이 보고서 미생성 세션(카드 AQ 표시 불가) 제외. 학습 중간에 나간 IN_PROGRESS 세션도 자연 배제 (실측: IN_PROGRESS 70~79·type NULL 61~63·중단 77 전부 제외 확인)
- sessionName = LEARNING_SESSION.SESSION_NAME (today=`오늘의 학습 - {테마명}` / theme=시나리오명 — 컨텐츠 확정 전은 테마 기반)
- createdAt은 **ISO 타임스탬프** 전달 — 표현(YYYY.mm.dd)은 **클라 포맷** 책임 (포맷은 presentation 관심사 — 서버는 CREATED_AT 원시값만). aq는 "AQ nn점"으로 클라 포맷
- 페이징 미도입 (발표 전 규모 — 성능 이슈 시 후속)
- 카드 터치 → 8.3 세부 보고서

### 8.3 GET /api/v1/sessions/{sessionId}/report — 세부 보고서 (상세) — ✅ **D-5 실구현 완료 (2026-09-06)**

```
GET /api/v1/sessions/{sessionId}/report?userId={userId}
```

⚠️ **인증 (v1.6 확정):** permitAll 경로 — **userId 쿼리파라미터 소유 검증** 필수 (세션 소유 유저만 조회 허용 — 타 유저 E0400). 클라는 기존 세션 플로우 임시 계약과 동일 패턴으로 userId 병용.

```json
// 응답 data — 실측 (세션 76)
{
  "sessionId": 76, "aq": 81,
  "totalFeedback": "전반적으로 안정적인 발화를 보였습니다...",
  "radar": { "listen": 100.0, "naming": 79.5, "shadowing": 70.0, "selfTalk": 74.5 },
  "metricCards": [
    { "type": "LISTEN", "score": 100.0, "feedback": "알아듣기 문제를 대부분 정확히 골랐어요. (평균 100점)",
      "turns": [
        { "turnId": 485, "turnNumber": 5, "promptText": "포도을(를) 고르세요",
          "ttsUrl": "/api/v1/voice/461", "imageUrl": null,
          "answer": { "mediaType": "image", "value": "94", "correct": true } },
        { "turnId": 487, "turnNumber": 7, "promptText": "여기서 금요일이 며칠인지 말해보세요",
          "ttsUrl": "/api/v1/voice/463", "imageUrl": null,
          "answer": { "mediaType": "text", "value": "8월 12일", "correct": true } }
      ] }
  ],
  "talkFeedback": "자유 대화에 적극적으로 참여하셨습니다...",
  "talkHistory": [
    { "speaker": "AI", "text": "수고하셨습니다. 마지막으로 오늘 기분이...", "ttsUrl": null },
    { "speaker": "USER", "text": "오늘은 카페에 갔어요", "voiceUrl": "/api/v1/voice/520" }
  ],
  "reportViewedAt": "2026-09-06T01:54:09.316230"
}
```

- radar = **해당 세션 TURN.score 집계** (대표점수 아님 — 대시보드 8.1과 출처 구분) — LISTEN=LISTEN_TEXT+LISTEN_PICTURE 통합 4축, score NULL 턴 제외, 해당 타입 턴 없으면 null
- metricCards[] — type(LISTEN|NAMING|SHADOWING|SELF_TALK) / score(해당 타입 평균=radar와 동일) / feedback(LEARNING_SESSION.*_feedback 간이 보고서 적재분) / turns(해당 타입 문제풀이 턴 배열)
- metricCards[].turns — 문제 안내(prompt_text)·AI TTS(ttsUrl)·내 답변(answer)·imageUrl(NAMING/SELF_TALK)
- **answer 계약 (v1.6 확정):** LISTEN_TEXT=`{mediaType:"text", value:"선택했던 선택지 텍스트", correct:bool}` — value는 choices_json 역직렬화로 1-based order에서 텍스트 추출(실측: "8월 12일") / LISTEN_PICTURE=`{mediaType:"image", value:"선택지 context(=image_id 문자열)", correct:bool}` (실측: "94") / 음성형=`{mediaType:"voice", value:"answer_text(STT)", voiceUrl}`
- LISTEN은 정답 여부(correct) 포함 / 음성형은 STT 텍스트(answer_text) + 유저 음성 voiceUrl 동봉
- talkHistory — AI 발화(text, VOICE_RECORD AI행 있으면 ttsUrl) + 유저 답변(voiceUrl 다시 듣기)
- 응답 수신 시 백엔드가 **REPORT_VIEWED_AT 기록** — **null일 때만 기록**(최초 1회, v1.6 구현 방식 확정 — 매 조회 덮어쓰기 아님). 응답 reportViewedAt = 기록된 시각(이미 기록돼 있으면 기존 시각 반환)
- 학습 중단 세션(COMPLETED_NO_TALK)은 8.2 리스트에서 제외 — 8.3 직접 호출도 **E0404** (실측)

## 9. 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-09-04 | 초안 — demo 브랜치 실구현 역추적 작성 (SessionFlowController/SessionFlowDtos/AiContainerClient/SecurityConfig/application.yml 실측) |


| v1.6 | 2026-09-06 | **D-5 대시보드+세션 플로우 실구현 전면 갱신:** §3.1 세션 2종 분기(POST /sessions/today·/theme — thema 쿼리파라미터 TEST/HOSPITAL/CAFE 이외 E0400·소문자 허용, /v2는 하위호환 유지, SessionCreateData.type 필드 신설, LISTEN 세분화 D-2 완료 표기), §3.5 finish 재편(간이 보고서 응답 — talk/total 항상 null·중단/완료 판정 규약·userAQ=REP_SCORES 캐시 조회 교체), §8.2 ✅ D-5 실구현(JWT 필수·STATUS != COMPLETED_NO_TALK **AND AQ IS NOT NULL** 규약 확정·페이징 미도입), §8.3 ✅ D-5 실구현(userId 쿼리파라미터 소유 검증·radar TURN 집계·answer 계약 확정 — LISTEN_TEXT=선택지 텍스트 추출·LISTEN_PICTURE=image_id context·REPORT_VIEWED_AT null일 때만 기록·중단 세션 E0404), §0 인증 현황 갱신(history JWT·report userId 병용), §6.1/6.2 전환 가이드 갱신(talk-turn-limit 8·계약 키 표 2종 엔드포인트+리포트 2단계). **클라 D-6 연동 시 주의: finish 응답 talk/total null — 세부 보고서는 §8.3에서 수령** |
| v1.5 | 2026-09-05 | **D-3 가입 플로우 API 실구현 반영:** §2 전면 갱신 — PATCH /me 확장(hobbies/sex/birthDate ISO/tagIds 전량 교체·>5개 E0400·없는 tag_id E0404·birthDate 파싱 실패 E0400), GET /me/tags 신설(15종 마스터), POST /me/survey 신설(서버 산출 환산 AQ 30/70/90 + REP_SCORES upsert — 중복 응답 허용), GET /me/scores 신설(§8.1 ⏳→실구현 전환), DELETE /me FK 역순 8단계 확장(USER_PROFILE_TAGS→REP_SCORES 추가, TAGS 마스터 보존). UserDto 확장 5필드(hobbies/sex/birthDate/tags/userAq — 하위호환 추가만), userAq null=설문 미응답 재노출 판별 기준 표기 |
| v1.4 | 2026-09-04 | **컨테이너 협의 확정 (7) 반영:** §8 신설(⏳ 구현 예정) — 대시보드/세부 보고서 API 3종(GET /users/me/scores 대표점수·GET /users/me/sessions/history 학습 카드(STATUS != COMPLETED_NO_TALK)·GET /sessions/{id}/report 세부 보고서). 방사형 출처 구분(대시보드=대표점수 캐시 / 세부=TURN 집계), REPORT_VIEWED_AT 갱신 연동, 학습 중단 세션 제외 |
| v1.1 | 2026-09-03 | B-1~B-3 수정 반영 — DELETE /me 204 확정(B-1 FK 역순 하드딜리트+OCI 정리), talk userText 스텁 더미 STT(B-2), §7 이슈 전건 해결 표기 |