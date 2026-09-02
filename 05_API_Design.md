# API 인터페이스 설계서 (API Interface Design — 클라이언트 ↔ Spring Boot)

> **버전:** v2.0 (2026-09-02 — 전면 리라이트, 문제풀이 기획 반영)
> **범위:** Android 클라이언트 ↔ Spring Boot API. **Spring Boot ↔ AI 컨테이너 계약은 `03_AI_Container_Contract.md`**
> **플로우 맥락:** `06_Session_Flow_Spec.md` · 스키마 = `04_Database_Design.md`

## 1. 기본 규격

### 1.1 Base URL

| 환경 | URL |
|------|-----|
| 개발/운영 | `http://{VM주소}` (:80 — nginx 리버스 프록시 경유, P3-23 확정) |

### 1.2 응답 형식 (성공)

```json
{ "success": true, "data": { ... }, "timestamp": "2026-09-02T12:34:56.789Z" }
```

### 1.3 응답 형식 (에러)

```json
{
  "success": false,
  "error": { "code": "E0101", "message": "유효하지 않은 요청입니다.", "detail": "session_id는 필수입니다.", "timestamp": "..." }
}
```

| 필드 | 설명 |
|------|------|
| `code` | `E{HTTP상태코드}{시퀀스}` |
| `message` | 사용자 노출용 한국어 메시지 |
| `detail` | 디버깅용 상세 |

### 1.3 JSON 네이밍

- **응답 JSON = camelCase** (Jackson 기본). 요청 DTO는 예외적으로 snake_case 일부 (`FirebaseAuthRequest.id_token` 등).

### 1.4 HTTP 상태 코드

| 코드 | 사용 |
|------|------|
| 200/201/204 | 성공 |
| 400 / 401 / 403 / 404 / 409 | 표준 |
| 500 | 서버 오류 |
| 502/504 | AI 컨테이너 오류/타임아웃 |

---

## 2. 인증 API (REST) — ✅ 구현 완료

| Method | Path | 설명 | 인증 |
|--------|------|------|------|
| POST | `/api/v1/auth/firebase` | Firebase ID Token 검증 → 자체 JWT 발급 | 불필요 |
| POST | `/api/v1/auth/refresh` | Access Token 갱신 | Refresh Token (`X-Refresh-Token`) |
| POST | `/api/v1/auth/logout` | 로그아웃 | Access Token |

**POST /auth/firebase 요청/응답:**

```json
// 요청
{ "id_token": "Firebase ID Token" }
// 응답
{ "success": true, "data": {
  "access_token": "...", "refresh_token": "...", "expires_in": 900,
  "user": { "id": 1, "uuid": "...", "email": "user@example.com", "nickname": null, "is_new_user": true }
} }
```

> `user.id` 필드는 Android 연동용으로 추가됨 (2026-09-01).

## 3. 사용자 API (REST) — ✅ 구현 완료

| Method | Path | 설명 | 상태 |
|--------|------|------|------|
| GET | `/api/v1/users/me` | 내 프로필 조회 | ✅ |
| PATCH | `/api/v1/users/me` | 프로필 수정 (닉네임) | ✅ |
| DELETE | `/api/v1/users/me` | 회원탈퇴 (DB hard delete + Firebase afterCommit) | ✅ |
| POST | `/api/v1/users/me/profile-image` | 프로필 사진 업로드 (multipart 5MB) | ✅ |
| GET | `/api/v1/users/me/profile-image` | 프로필 사진 조회 (백엔드 프록시 스트리밍) | ✅ |
| GET | `/api/v1/users/me/level` | 유저 수준/대시보드 조회 | 🔜 미구현 |

- 회원탈퇴: 클라이언트는 Firebase `.delete()` + 로컬 토큰 삭제 후 로그인 화면 이동.
- 프로필 사진: OCI 비공개 버킷 + 서버 스트리밍, 캐시 무효화는 `?v=` 쿼리.
- **신규 유저 판별:** `isNewUser == true || nickname == null` → SignUpActivity.

## 4. 세션/학습 API (REST) — 🔜 신규 구현 대상

> 기획 기준: `06_Session_Flow_Spec.md`. 이번 개발 단위의 핵심.
> ⚠️ dev용 임시 규칙: `SecurityConfig`에서 `/api/v1/sessions`, `/api/v1/voice/**` permitAll (운영 반영 금지).

### 4.1 세션 생성 — "오늘의 학습" 시작

| 항목 | 내용 |
|------|------|
| **Method** | `POST` |
| **Path** | `/api/v1/sessions` |
| **인증** | Bearer JWT |

**동작:** 테마 랜덤 선택(TEST/HOSPITAL/CAFE) → LEARNING_SESSION INSERT → IMAGE_THEMA로 테마 이미지 풀 조회 → **AI 컨테이너 `POST /sessions`(8문제 일괄 생성)** → TURN 8행 INSERT(PENDING) + TTS OCI 적재 + VOICE_RECORD AI 행 → 응답.

#### 응답

```json
{
  "success": true, "data": {
    "sessionId": 101,
    "theme": "HOSPITAL",
    "turns": [
      {
        "turnId": 501,
        "turnNumber": 1,
        "type": "LISTEN",
        "ttsUrl": "/api/v1/voice/9001",
        "passage": "여기서 금요일이 며칠인지 말해보세요",
        "choices": [
          { "order": 1, "mediaType": "TEXT",  "context": "8월 12일" },
          { "order": 2, "mediaType": "TEXT",  "context": "8월 20일" }
        ]
      },
      {
        "turnId": 502, "type": "NAMING", "imageId": 33,
        "imageUrl": "/api/v1/content/images/33",
        "hintAvailable": 2
      }
    ]
  }
}
```

> 문제 생성에 수십 초 가능 → 클라는 로딩 화면 표시 (기획 확정: 전부 선생성 + 로딩 대기).

### 4.2 세션 상태/이어하기

| 항목 | 내용 |
|------|------|
| **Method** | `GET` |
| **Path** | `/api/v1/sessions` (목록) / `/api/v1/sessions/{id}` (상세) |
| **설명** | 세션 목록 = 학습 이력 화면. 상세 = TURN.status(PENDING/SUBMITTED/SCORED) 기반 진행 위치 → 이어하기/폐기·신규 판별 |

### 4.3 턴 답안 제출 (타입별)

> 공통 플로우: 답안 제출 → TURN UPDATE(status=SUBMITTED, selected_value/answer_text) → 채점(아래 방식) → 결과 적재(score, VOICE_RECORD) → status=SCORED. **클라는 결과를 기다리지 않고 다음 턴 진행 가능** (채점 결과는 세션 결과 화면에서 일괄 확인).

| 타입 | Method/Path | 요청 | 채점 방식 |
|------|-------------|------|-----------|
| LISTEN | `POST /api/v1/sessions/{id}/turns/{turnId}/listen` | `{ "selected": "<choice ref>" }` | **백엔드 자체** — 정답 100 / 오답 0. 즉시 응답 |
| NAMING | `POST /api/v1/sessions/{id}/turns/{turnId}/naming` | 음성 multipart (+ 힌트 사용 후라면 횟수) | 컨테이너 `/answer/naming` 호출 |
| SHADOWING | `POST /api/v1/sessions/{id}/turns/{turnId}/shadowing` | 음성 multipart | 컨테이너 `/answer/shadowing` |
| SELF_TALK | `POST /api/v1/sessions/{id}/turns/{turnId}/selftalk` | 음성 multipart | 컨테이너 `/answer/selftalk` |

> 유저 음성: multipart 업로드 → 백엔드가 즉시 OCI 적재 + 공유폴더 사본 → 컨테이너에 `userVoicePath` 전달. STT 텍스트(`text`)와 발화지표는 컨테이너 응답(`userVoiceEval`)으로 수신 적재.
>
> NAMING의 `userRT`는 백엔드가 VOICE_RECORD에서 산정 (최근 naming 20개 중 발화시간/음절수 최단 10개, 0개면 null).

### 4.4 NAMING 힌트 요청

| 항목 | 내용 |
|------|------|
| **Method** | `POST` |
| **Path** | `/api/v1/sessions/{id}/turns/{turnId}/hint` |
| **인증** | Bearer JWT |
| **동작** | 의미단서→조음단서 순서로 1개 반환 (동기), `hints_shown` 증가, 감점은 컨테이너 채점 시 `hintCount`로 반영 |

```json
{ "success": true, "data": { "hintOrder": 1, "cueType": "SEMANTIC", "text": "컴퓨터를 조작할 때 손에 쥐는 것" } }
```

### 4.5 이야기 턴 (STORYTELLING)

| 항목 | 내용 |
|------|------|
| **Method** | `POST` |
| **Path** | `/api/v1/sessions/{id}/turns/talk` |
| **동작** | (1) 첫 호출: 컨테이너 `/aichat`(context 빈 배열 + turnResults + userInfos) → AI 첫 대사 수신 → TURN INSERT(prompt_text) · (2) 이후: 유저 음성 multipart + 최신 context → 컨테이너 → `llmResponse` + `userText` 수신 → TURN 기록 |
| **채점** | 없음. 발화지표 없음 (VOICE_RECORD USER 행 생성, 3종 NULL) |
| **종료** | 조기종료 버튼 / 8턴 하드캡 (AI 마지막 응답으로 마무리) |

### 4.6 세션 종료 + 리포트

| 항목 | 내용 |
|------|------|
| **Method** | `POST` |
| **Path** | `/api/v1/sessions/{id}/finish` |
| **동작** | 세션 종료 처리 → 컨테이너 `POST /report` (turns + talkContext) → sessionAQ + 피드백 6종 수신 → LEARNING_SESSION UPDATE(COMPLETED) → **동기 응답** (클라는 로딩 대기) |

```json
{ "success": true, "data": {
  "sessionAQ": 82,
  "feedbacks": {
    "listenFeedback": "...", "namingFeedback": "...", "shadowingFeedback": "...",
    "selfTalkFeedback": "...", "talkFeedback": "...", "totalFeedback": "..."
  }
} }
```

> 생성 실패 시 재시도는 고도화 과제 (pending).

### 4.7 음성 스트리밍 (재생)

| 항목 | 내용 |
|------|------|
| **Method** | `GET` |
| **Path** | `/api/v1/voice/{voiceRecordId}` |
| **인증** | Bearer JWT |
| **동작** | 백엔드 프록시 스트리밍 (OCI 원본). 유저 m4a / AI mp3 모두. Content-Type: `audio/*` |

### 4.8 콘텐츠 리소스 (이미지)

| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/v1/content/images` | 이미지 목록 (테마 필터 — IMAGE_THEMA 조인) |
| GET | `/api/v1/content/images/{id}` | 이미지 상세 — 경로(image_tag_path/semantic_cue/articulatory_cue) 반환 |
| GET | `/api/v1/content/images/{id}/file` | 이미지 파일 프록시 스트리밍 (PAR 불필요한 BE 경유안) |

> 태그 JSON은 셀/클라에 노출하지 않고 백엔드가 컨테이너 채점 요청에만 전달.

## 5. WebSocket — 상태

기획 전환(문제풀이 앱) 이후 턴 진행은 **REST 폴링/동기 제출 기반**으로 단순화. WebSocket(TURN_RESULT 푸시 등)은 pending — 턴당 채점 수신 방식 상의(진행 중) 결과에 따라 재설계. 기존 WebSocketManager 코드는 유지.

## 6. 구현 상태 요약

| API | 상태 |
|-----|------|
| 인증 3종 / 사용자 5종 | ✅ 구현 완료 |
| 음성 업로드(P3-19: POST /api/v1/voice/upload) | ✅ 구현 완료 (E2E 검증) |
| 세션 생성/문제 일괄 생성 | 🔜 이번 단위 |
| 턴 답안 제출 4종 / 힌트 / 이야기 턴 | 🔜 이번 단위 |
| 세션 종료+리포트 / 음성 스트리밍 | 🔜 이번 단위 |
| 대시보드/유저수준/설정 API | ⏳ Phase 4 |

## 7. 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v2.0 | 2026-09-02 | 전면 리라이트 — 문제풀이 세션 API 신규 정의(§4 전체), WebSocket 상태 정리(§5), 음성 스트리밍/콘텐츠 API 갱신. 구 WebSocket 중심 설계는 archive로 |
| (구 v1.x) | ~2026-09-01 | 기본 규격/구현 완료 API 이력은 유지(§1~3 흡수) |