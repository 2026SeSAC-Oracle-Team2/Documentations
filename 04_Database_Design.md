# 데이터베이스 설계서 (Database Design Document)

> **버전:** v2.0 (2026-09-02 현행화 — 전면 리라이트)
> **기준:** LIVE DB (부트캠프 VM, Oracle XE 21c, PDB=XEPDB1) + 확정 기획
> **관련:** 세션/기획 = `06_Session_Flow_Spec.md` · 계약 = `03_AI_Container_Contract.md`
> **ADR:** ADR-003(리스너 자체채점) / ADR-004(힌트 DB화) / ADR-007(TURN.status) / ADR-008(REPORT 폐지) / ADR-010(발화지표 rename)
>
> ⚠️ **init SQL 동기화 안내:** `baseworks/Project/containers/SeSAC_SpeechApp_Container_DB/init/`의 일부 SQL은 LIVE DB 마이그레이션 전 상태(learning_session rename·SEQUENCE·TURN.score 등은 LIVE에 적용 완료, init SQL 반영은 DB 세션 작업). 본 문서가 **설계 기준**이며, 상충 시 본 문서와 LIVE DB가 우선.

## 1. 스키마 분리 구조

| 스키마 | 도메인 | 테이블 |
|--------|--------|--------|
| `SPEECHAPP_USER` | 사용자 + 학습 | APP_USER, USER_PROFILE, CONTENT_TYPE, LEARNING_SESSION, TURN, TURN_IMAGE, VOICE_RECORD |
| `SPEECHAPP_CONTENT` | 콘텐츠 | IMAGE_RESOURCE, IMAGE_THEMA |

## 2. DB 사용자 계정

| 계정 | SPEECHAPP_USER | SPEECHAPP_CONTENT | 용도 |
|------|----------------|-------------------|------|
| `SPEECHAPP_APP` | RW | RO (+IMAGE_RESOURCE SELECT/REFERENCES, 시퀀스 SELECT) | Spring Boot 애플리케이션 |
| `SPEECHAPP_ADMIN` | RW | RW | 관리자 페이지 / 마이그레이션 |

> ⚠️ **권한 함정 (2026-09-02 발견):** 신규 테이블 생성 시 `04_permissions.sql`의 GRANT가 LIVE DB에 자동 적용되지 않음 — 수동 GRANT 필수 (미적용 시 ORA-01031).

---

## 3. ERD (현행)

```
SPEECHAPP_USER
  APP_USER (id PK) 1──1 USER_PROFILE (user_id FK, UNIQUE)
  APP_USER 1──N LEARNING_SESSION (user_id FK)
  LEARNING_SESSION 1──N TURN (session_id FK)
  TURN N──N IMAGE_RESOURCE ← TURN_IMAGE (turn_id+image_id 복합 PK)
  TURN 1──N VOICE_RECORD (turn_id FK, UNIQUE(turn_id, speaker))
  CONTENT_TYPE (type_code 자연키 PK) 1──N TURN (content_type FK)

SPEECHAPP_CONTENT
  IMAGE_RESOURCE (image_id PK) 1──N IMAGE_THEMA (image_id FK)
```

---

## 4. 테이블 상세 — SPEECHAPP_USER

### 4.1 APP_USER (사용자 계정)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `ID` | NUMBER(19) | PK, IDENTITY | |
| `UUID` | VARCHAR2(36) | NOT NULL, UNIQUE | 내부 식별자 (버킷 경로 조립용) |
| `FIREBASE_UID` | VARCHAR2(128) | NOT NULL, UNIQUE | nullable로 확장 가능 (소셜 확장 대응 — 현행 NOT NULL) |
| `EMAIL` | VARCHAR2(255) | NOT NULL, UNIQUE | |
| `SOCIAL_PROVIDER` | VARCHAR2(20) | | |
| `SOCIAL_ID` | VARCHAR2(255) | | |
| `CREATED_AT` / `UPDATED_AT` | TIMESTAMP | NOT NULL | |

### 4.2 USER_PROFILE (사용자 프로필)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `ID` | NUMBER(19) | PK, IDENTITY | |
| `USER_ID` | NUMBER(19) | NOT NULL, UNIQUE, FK→app_user | |
| `NICKNAME` | VARCHAR2(50) | | |
| `PROFILE_IMAGE_BUCKET_PATH` | VARCHAR2(500) | | OCI key (`{userUUID}/profile.{ext}`) |
| `LIKES` 🆕 | VARCHAR2(?) | nullable | 관심사 태그 — AI 컨테이너 userInfos 전달용 |
| `SEX` 🆕 | VARCHAR2(?) | nullable | 성별 |
| `AGE` 🆕 | NUMBER | nullable | 나이 |
| `CREATED_AT` / `UPDATED_AT` | TIMESTAMP | NOT NULL | |

> 🆕 3종(likes/sex/age)은 2026-09-02 확정 — AI 컨테이너의 `userInfos`에 전달되어 문제 출제·대화 개시 맥락으로 사용. **정확한 크기/자료형은 DB 세션에서 DDL 확정 시 결정.** JPA Entity(UserProfile.kt)도 동기화 필요.

### 4.3 CONTENT_TYPE (컨텐츠 타입 룩업)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `TYPE_CODE` | VARCHAR2(20) PK (자연키) | LISTEN / NAMING / SHADOWING / SELF_TALK / STORYTELLING |
| `TYPE_NAME` | VARCHAR2(50) | 듣기 / 이름 맞추기 / 쉐도잉 / 자기 대화 / 스토리텔링 |
| `CATEGORY` | VARCHAR2(50) | receptive / productive (기획상 A=채점 4종, B=무채점 1종) |

> TURN의 CHECK 제약이 5종 코드를 강제. STORYTELLING은 채점 없음 — `TURN.score` 항상 NULL.

### 4.4 LEARNING_SESSION (학습 세션)

> ⚠️ Oracle 예약어 `SESSION` 회피 → `learning_session` rename + `SESSION_SEQ` 사용 (ORA-00931 함정, ADR 참고).

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `ID` | NUMBER(19) | PK, **SESSION_SEQ** | |
| `USER_ID` | NUMBER(19) | NOT NULL, FK→app_user | |
| `THEME` | VARCHAR2(30) | | 서버 랜덤 선택 (TEST/HOSPITAL/CAFE) |
| `STATUS` | VARCHAR2(20) | DEFAULT 'IN_PROGRESS' | IN_PROGRESS / COMPLETED / ... |
| `AQ` 🆕 | NUMBER(3) | nullable, CHECK(0~100) | **세션 총점** — 100점 만점 정수(소수점 올림은 AI 컨테이너 책임). 리포트 생성 시점에 적재. 리포트 전 NULL |
| `LISTEN_FEEDBACK` 🆕 | CLOB | nullable | 알아듣기 지표 피드백 |
| `NAMING_FEEDBACK` 🆕 | CLOB | nullable | 이름대기 지표 피드백 |
| `SHADOWING_FEEDBACK` 🆕 | CLOB | nullable | 따라말하기 지표 피드백 |
| `SELF_TALK_FEEDBACK` 🆕 | CLOB | nullable | 스스로말하기 지표 피드백 |
| `TALK_FEEDBACK` 🆕 | CLOB | nullable | AI 대화(STORYTELLING) 피드백 |
| `TOTAL_FEEDBACK` 🆕 | CLOB | nullable | 종합 피드백 |
| `CREATED_AT` / `UPDATED_AT` | TIMESTAMP | NOT NULL | |

> REPORT 테이블은 **신설하지 않음** (ADR-008) — 리포트 내용이 세션 행에 통합됨. 턴별 상세는 TURN 조회로 충분.

### 4.5 TURN (턴 / Q&A 1세트)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `ID` | NUMBER(19) | PK, TURN_SEQ | |
| `SESSION_ID` | NUMBER(19) | NOT NULL, FK→learning_session | |
| `TURN_NUMBER` | NUMBER(5) | NOT NULL | 세션 내 순서 |
| `CONTENT_TYPE` | VARCHAR2(20) | NOT NULL, FK→content_type, CHECK IN 5종 | |
| `STATUS` 🆕 | VARCHAR2(20) | | **PENDING**(출제·미풀이) / **SUBMITTED**(답안 제출) / **SCORED**(채점 완료) |
| `PROMPT_TEXT` | CLOB | | AI 제시 텍스트 (TTS 지문 / STORYTELLING AI 발화) |
| `CHOICES_JSON` | CLOB | LISTEN 필수 | `[{order, media_type: IMAGE\|TEXT, ref}]` |
| `CORRECT_VALUE` | VARCHAR2(255) | LISTEN·NAMING 필수 | LISTEN=정답 choice ref / NAMING=정답 단어 / SHADOWING=원문 / SELF_TALK=**NULL** |
| `SELECTED_VALUE` | VARCHAR2(255) | | LISTEN=유저가 탭한 choice ref, 그 외 NULL |
| `ANSWER_TEXT` | CLOB | | 유저 발화 STT (NAMING/SHADOWING/SELF_TALK/STORYTELLING) |
| `HINTS_SHOWN` | NUMBER(1) | | NAMING 힌트 노출 수 (0~2) |
| `SCORE` (v1.95 추가) | NUMBER(5,2) | nullable, 0~100 | **평가지표 채점 결과** — STORYTELLING 항상 NULL |
| `CREATED_AT` | TIMESTAMP | NOT NULL | |

**CHECK 제약:** LISTEN → choices_json·correct_value NOT NULL / NAMING → correct_value NOT NULL.

**타입별 컬럼 사용 규격:**

| 타입 | prompt_text | choices_json | correct_value | selected_value | answer_text |
|------|-------------|--------------|---------------|----------------|-------------|
| LISTEN | TTS 지문 | 선택지 2~4개 | 정답 ref | 유저 선택 ref | — (탭 선택) |
| NAMING | (출제 멘트) | — | 정답 단어 | NULL | STT 발화 |
| SHADOWING | 원문 구문 | — | 원문 | NULL | STT 발화 |
| SELF_TALK | (출제 멘트) | — | NULL | NULL | STT 발화 |
| STORYTELLING | AI 발화 | — | NULL | NULL | STT 발화 |

### 4.6 TURN_IMAGE (턴-이미지)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `TURN_ID` + `IMAGE_ID` | 복합 PK | cross-schema FK (REFERENCES grant 필요) |
| `IMAGE_ORDER` | NUMBER(2) | UNIQUE(turn_id, image_order) — LISTEN 선택지 순서 |

### 4.7 VOICE_RECORD (음성 녹음 메타데이터)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `ID` | NUMBER(19) | PK, VOICE_RECORD_SEQ | |
| `USER_ID` / `SESSION_ID` / `TURN_ID` | NUMBER(19) | FK 각각 | |
| `SPEAKER` | VARCHAR2(10) | CHECK IN('USER','AI') | |
| `VOICE_FILE_PATH` | VARCHAR2(500) | NOT NULL | OCI key: `containers/llm/{userUUID}/{sessionID}/{turnID}_user.m4a\|_ai.mp3` |
| `DURATION_SECONDS` | NUMBER(5) | | 녹음 시간 |
| `SYLLABLES` | NUMBER | nullable | **발화지표 1** 음절 수 |
| `SPEAKING_TIME` 🆕 (구 RESPONSE_TIME) | NUMBER | nullable | **발화지표 2** 발화시간 — 정의는 계약서 기준 |
| `ARTICULATION_TIME` 🆕 (구 ARTICULATION_RATE) | NUMBER | nullable | **발화지표 3** 조음시간 — 정의는 계약서 기준 |
| `CREATED_AT` | TIMESTAMP | NOT NULL | |

**제약:** UNIQUE(turn_id, speaker) / AI행은 발화지표 3종 NULL 강제 CHECK (rename 후 컬럼명 갱신 필요).

> **rename 근거 (ADR-010):** 측정 지표가 변경됨 (RESPONSE_TIME→SPEAKING_TIME, ARTICULATION_RATE→ARTICULATION_TIME). 값 있는 레코드 0건이라 rename 무부담. STORYTELLING은 발화지표 미기록 — 유저 행 생성 시 3종 NULL.

---

## 5. 테이블 상세 — SPEECHAPP_CONTENT

### 5.1 IMAGE_RESOURCE (이미지 리소스)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `IMAGE_ID` | NUMBER(19) | PK, IDENTITY | |
| `IMAGE_NAME` | VARCHAR2(200) | NOT NULL | 이미지 표시명 (NAMING 정답 단어 소스) |
| `IMAGE_FILE_PATH` | VARCHAR2(500) | NOT NULL | OCI key: `images/{id}/{id}.jpg` |
| `IMAGE_TAG_PATH` | VARCHAR2(500) | nullable | OCI key: `images/{id}/{id}.tags.json` — **SELF_TALK 채점용 태그** |
| `SEMANTIC_CUE` 🆕 | VARCHAR2(?) | nullable | **의미단서** — NAMING 힌트 1순위 |
| `ARTICULATORY_CUE` 🆕 | VARCHAR2(?) | nullable | **조음단서** — NAMING 힌트 2순위 |
| `CREATED_AT` | TIMESTAMP | NOT NULL | |

> **v2.0 변경 (ADR-004):** `IMAGE_HINT_PATH` 폐지 + OCI `hint.json` 규약 폐지 → 힌트를 DB 컬럼으로 저장. cue는 NAMING 전용이라 nullable (SELF_TALK 이미지는 미입력). 힌트 공개 API는 파일 파싱 없이 컬럼 조회.
> **OCI 규약 (최종 2종):** `images/{id}/{id}.jpg` + `images/{id}/{id}.tags.json`
> **관리자 페이지 파급:** 힌트 JSON 첨부 → cue 텍스트 2필드로 변경, 행 삭제 시 OCI 연동 삭제 대상에서 hint.json 제외.

### 5.2 IMAGE_THEMA (이미지↔테마 매핑)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `ID` | NUMBER(19) | PK, IDENTITY | |
| `IMAGE_ID` | NUMBER(19) | NOT NULL, FK→image_resource | |
| `THEMA_KEY` | VARCHAR2(30) | CHECK IN('TEST','HOSPITAL','CAFE') | |
| | | UNIQUE(image_id, thema_key) | 한 이미지가 여러 테마에 속할 수 있음 |

> 세션 테마가 랜덤 결정되면 이 테이블로 해당 테마의 사용 가능한 이미지 풀을 조회 → `/sessions`의 `imageList` 전달. 룩업 테이블 없이 key 문자열 방식 (CONTENT_TYPE 선례).

---

## 6. 데이터 흐름 (DB 관점)

```
[세션 생성] LEARNING_SESSION INSERT(IN_PROGRESS)
  → POST /sessions → problemList 수신 → TURN 8행 INSERT(PENDING)
  → AI TTS → VOICE_RECORD AI 행 + OCI 적재
[답안 제출] TURN UPDATE(selected_value/answer_text, status=SUBMITTED)
  → 유저 음성 OCI 적재 + VOICE_RECORD USER 행(발화지표 3종 포함, LISTEN 제외)
  → 채점 완료 → TURN.score 저장, status=SCORED
[이야기 턴] TURN 1행씩 INSERT(prompt_text=AI, answer_text=유저)
  VOICE_RECORD USER 행은 생성하되 발화지표 3종 NULL
[세션 종료] POST /report → AQ + 피드백 6종 → LEARNING_SESSION UPDATE(COMPLETED)
```

**주요 조회:**
- 지표별 실력: 최근 20세션 중 지표별 상위 10개 세션 평균 (ADR-009)
- 이어하기: TURN.status로 진행 위치 판별
- 리포트: TURN(SESSION_ID) 집계 + LEARNING_SESSION 피드백 컬럼

## 7. 시퀀스 / JPA 매핑 규칙

| 항목 | 내용 |
|------|------|
| SEQUENCE | session_seq / turn_seq / voice_record_seq — Oracle 예약어·IDENTITY 함정 회피 (learning_session rename 참고) |
| 소수 컬럼 | **반드시 BigDecimal + precision/scale** — Kotlin `Double`은 Hibernate가 binary_double로 매핑되어 NUMBER(x,y) validation 실패 (TURN.score 사례) |
| Timestamp | TIMESTAMP + `@CreationTimestamp`/`@UpdateTimestamp` |
| 스키마 | `@Table(schema="speechapp_user"|"speechapp_content")` |
| 권한 | SEQUENCE SELECT 권한 필수 — 미부여 시 ORA-00942 |

## 8. 데이터 관리 정책

| 항목 | 정책 |
|------|------|
| 테스트 프로브 유저 | `probe-dev@example.test` — 운영 전 클렌징 (개인 dev VM 잔존분) |
| 태그 보유 이미지 | tag 레코드 존재 — tags.json 관련 로직 수정 시 데이터 보존 유의 |
| hint.json | 보유 레코드 0건 — SEMANTIC/ARTICULATORY_CUE 전환 시 마이그레이션 불필요 |
| VOICE_RECORD rename | 값 있는 레코드 0건 — rename 무부담 |
| 콘텐츠 시딩 | 관리자 페이지로 (이미지 필수 + 태그 JSON 선택) |

## 9. 미구현 (채점 확정 후 대기)

| 항목 | 상태 |
|------|------|
| `TURN.score` 저장 로직 (백엔드) | 계약 구현과 함께 |
| 유저 수준/대시보드 집계 쿼리 | 실력 산정식 확정으로 구현 가능 |
| init SQL 05번 → 현행 기준 동기화 | DB 세션 작업 |

## 10. 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v2.0 | 2026-09-02 | **전면 리라이트** — 폐지 테이블/이력 제거, 현행 스키마만 기술. DB 수정 확정사항 반영: IMAGE_RESOURCE cue 2컬럼(IMAGE_HINT_PATH 폐지), LEARNING_SESSION AQ+피드백 6컬럼, VOICE_RECORD rename(SPEAKING_TIME/ARTICULATION_TIME), USER_PROFILE likes/sex/age, TURN.status. OCI hint.json 규약 폐지 |
| (구 v0.5 이하) | ~2026-09-01 | 세션/턴/녹음 재설계 이력은 archive 및 계획서 §15 참고 |