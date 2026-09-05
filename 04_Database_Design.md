# 데이터베이스 설계서 (Database Design Document)

> **버전:** v2.6 (2026-09-04 현행화)
> **기준:** LIVE DB (부트캠프 VM, Oracle XE 21c, PDB=XEPDB1) + 확정 기획
> **관련:** 세션/기획 = `06_Session_Flow_Spec.md` · 계약 = `03_AI_Container_Contract.md`
> **ADR:** ADR-003(리스너 자체채점) / ADR-004(힌트 DB화) / ADR-007(TURN.status) / ADR-008(REPORT 폐지) / ADR-010(발화지표 rename)
>
> ⚠️ **init SQL 동기화 안내:** `baseworks/Project/containers/SeSAC_SpeechApp_Container_DB/init/`의 일부 SQL은 LIVE DB 마이그레이션 전 상태(learning_session rename·SEQUENCE·TURN.score 등은 LIVE에 적용 완료, init SQL 반영은 DB 세션 작업). 본 문서가 **설계 기준**이며, 상충 시 본 문서와 LIVE DB가 우선.

## 1. 스키마 분리 구조

| 스키마 | 도메인 | 테이블 |
|--------|--------|--------|
| `SPEECHAPP_USER` | 사용자 + 학습 | APP_USER, USER_PROFILE, **TAGS, USER_PROFILE_TAGS**(v2.2), **USER_REPRESENTATIVE_SCORES**(v2.6), CONTENT_TYPE, LEARNING_SESSION, TURN, TURN_IMAGE, VOICE_RECORD |
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
  APP_USER (id PK) 1──1 USER_PROFILE (user_id FK, UNIQUE: profile_image/nickname/hobbies/sex/birth_date/user_memory) 1──1 USER_REPRESENTATIVE_SCORES (user_id FK UNIQUE: user_aq/user_score_listen/user_score_naming/user_score_shadowing/user_score_self_talk)
  APP_USER 1──N USER_PROFILE_TAGS N──1 TAGS (관심사 태그, 최대 5개/유저)
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

### 4.2 USER_PROFILE (사용자 프로필) — v2.2 가입 플로우 개편

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `ID` | NUMBER(19) | PK, IDENTITY | |
| `USER_ID` | NUMBER(19) | NOT NULL, UNIQUE, FK→app_user | |
| `NICKNAME` | VARCHAR2(50) | | |
| `PROFILE_IMAGE_BUCKET_PATH` | VARCHAR2(500) | | OCI key (`{userUUID}/profile.{ext}`) |
| `HOBBIES` 🆕 (구 LIKES rename) | VARCHAR2(500) | nullable | 취미 자유 텍스트 — 가입 플로우에서 유저 입력 |
| `SEX` | VARCHAR2(10) | nullable | 성별 |
| `BIRTH_DATE` 🆕 (구 AGE 대체) | DATE | nullable | 생년월일 — 컨테이너 전달 시 `age`로 백엔드가 산정 |
| `USER_MEMORY` 🆕 (v2.2) | CLOB | nullable | **누적 개인화 메모리** — AI 대화 기반 컨테이너 관리 오파크 텍스트. 백엔드 파싱 비관여, 통째 전달/저장. 갱신 = /report/total 응답 1곳. 하드캡 8KB(방어용). 회원탈퇴 시 PROFILE 삭제에 자동 포함. 규약: 03a §10 |
| `CREATED_AT` / `UPDATED_AT` | TIMESTAMP | NOT NULL | |

> **v2.2 (2026-09-04, 컨테이너 협의):** `LIKES`→`HOBBIES` rename, `AGE`(NUMBER) 폐지 → `BIRTH_DATE`(DATE) 신설 — 나이 산정식 변경(백엔드), **`USER_MEMORY` CLOB 신설**. 구 3종 컬럼 설명(2026-09-02) 대체. JPA Entity(UserProfile.kt) 동기화 필요.
> **v2.6 (2026-09-04, 컨테이너 협의 5→7):** **대표점수를 USER_PROFILE에서 `USER_REPRESENTATIVE_SCORES` 전용 테이블로 이동** (§4.2.2 — USER_PROFILE 비대화 방지: 대표점수는 실력 데이터라 프로필 도메인·갱신 주기가 다름). 캐시 5종(AQ+4지표) 통합 저장 — AQ만 프로필에 두고 4지표가 분리되면 값이 찢어지는 문제 회피. 갱신 지점 고정(설문 접수 + /report/problems 수신)이라 정합성 논리는 v2.5와 동일. /sessions userAQ 전달은 USER_PROFILE LEFT JOIN 1개 추가 (여전히 단일 왕복). 대안 기각 이력: ① SURVEY_AQ+USER_AQ 2컬럼 ② LEARNING_SESSION 가짜 레코드. ⚠️ 마이그레이션 시 기존 세션 보유 계정 백필 필요(재노출 오검출 방지)

### 4.2.1 TAGS / USER_PROFILE_TAGS (관심사 태그 — v2.2 신설)

가입 플로우의 선택형 관심사 태그. 클라는 TAGS 테이블을 받아 버블 나열 → 최대 5개 선택.

| 테이블 | 컬럼 | 제약 | 설명 |
|--------|------|------|------|
| `TAGS` | `TAG_ID` NUMBER(19) PK IDENTITY / `TAG` VARCHAR2(50) NOT NULL UNIQUE | — | 태그 마스터 15종 시드: 건강관리·등산·골프·여행·트로트·요리·텃밭가꾸기·낚시·독서·바둑·사진·전시관람·국내여행·반려동물·봉사활동 |
| `USER_PROFILE_TAGS` | `USER_ID` FK→app_user + `TAG_ID` FK→tags (복합 PK) | 최대 5개는 앱 레벨 검증 | 유저↔태그 N:M |

> 컨테이너 전달 시: 선택 태그를 쉼표 구분 문자열(`"등산, 골프, 요리"`)로 조립해 userInfos.tags로 통째 전달 (03a §1.1). TAGS 시드는 init SQL 02번에 포함 — LIVE DB 마이그레이션 필요.

### 4.2.2 USER_REPRESENTATIVE_SCORES (대표점수 캐시 — v2.6 신설)

> 대표점수 5종(AQ+4지표)의 캐시 테이블. 대시보드 방사형 그래프 + /sessions userAQ 전달용.

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `USER_ID` | NUMBER(19) | PK, UNIQUE, FK→user_profile.user_id | 1:1 |
| `USER_AQ` | NUMBER(3) | nullable, CHECK(0~100) | **유저 대표 AQ** — 가입 설문 환산값(30/70/90) 초기 세팅 → /report/problems 수신 시점마다 ADR-009 식 재계산 UPDATE. **null = 설문 미응답** (설문 재노출 판별 기준) |
| `USER_SCORE_LISTEN` | NUMBER(5,2) | nullable | 대표 지표점수 — LISTEN_TEXT/LISTEN_PICTURE **통합** (지표상 동일 LISTEN). ADR-009 식: 최근 20세션 중 상위 10개 세션 평균 |
| `USER_SCORE_NAMING` | NUMBER(5,2) | nullable | 동일식 |
| `USER_SCORE_SHADOWING` | NUMBER(5,2) | nullable | 동일식 |
| `USER_SCORE_SELF_TALK` | NUMBER(5,2) | nullable | 동일식 |
| `UPDATED_AT` | TIMESTAMP | NOT NULL | |

> **갱신 지점 2곳 고정 (정합성):** ① 가입 설문 접수 — USER_AQ 초기 세팅(30/70/90) ② /report/problems 수신 — AQ+4지표 재계산 UPDATE. /sessions 요청 시 USER_PROFILE LEFT JOIN으로 userAQ 조회(단일 왕복). **방사형 그래프 출처 구분: 대시보드 = 이 테이블 / 세부 보고서 = 해당 세션 TURN.score 집계.**

### 4.3 CONTENT_TYPE (컨텐츠 타입 룩업)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `TYPE_CODE` | VARCHAR2(20) PK (자연키) | LISTEN_TEXT / LISTEN_PICTURE / NAMING / SHADOWING / SELF_TALK / STORYTELLING — **v2.3: LISTEN 세분화** (구 LISTEN 폐지 — 다시보기 구분·난이도 운영용) |
| `TYPE_NAME` | VARCHAR2(50) | 듣기(텍스트) / 듣기(그림) / 이름 맞추기 / 쉐도잉 / 자기 대화 / 스토리텔링 |
| `CATEGORY` | VARCHAR2(50) | receptive / productive (기획상 A=채점 4종, B=무채점 1종) |

> TURN의 CHECK 제약이 6종 코드를 강제 (v2.3: LISTEN → LISTEN_TEXT/LISTEN_PICTURE). STORYTELLING은 채점 없음 — `TURN.score` 항상 NULL. LISTEN_TEXT/LISTEN_PICTURE는 지표상 모두 LISTEN (운영 구분용 — 채점 방식 동일, 백엔드 자체 100/0).

### 4.4 LEARNING_SESSION (학습 세션)

> ⚠️ Oracle 예약어 `SESSION` 회피 → `learning_session` rename + `SESSION_SEQ` 사용 (ORA-00931 함정, ADR 참고).

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `ID` | NUMBER(19) | PK, **SESSION_SEQ** | |
| `USER_ID` | NUMBER(19) | NOT NULL, FK→app_user | |
| `THEME` | VARCHAR2(30) | | 서버 랜덤 선택 (TEST/HOSPITAL/CAFE) |
| `SESSION_NAME` 🆕 (v2.6) | VARCHAR2(100) | | **세션 표시명** — 학습 기록 카드용. today=`오늘의 학습 - {테마명}` / theme=`병원에서 진료받기`·`동네카페에서`(시나리오 플로우 고정 시퀀스 — 컨텐츠 팀 확정분) |

| `TYPE` 🆕 (v1.3) | VARCHAR2(20) | | 세션 종류 — `today`(오늘의 학습: 테마 랜덤+무작위 출제) / `theme`(테마별 학습: 기획 시나리오 플로우). 컨테이너 엔드포인트 분기(/sessions/today vs theme)와 매핑 |
| `STATUS` | VARCHAR2(20) | DEFAULT 'IN_PROGRESS' | IN_PROGRESS / COMPLETED / **COMPLETED_NO_TALK**(이야기 턴 없이 조기종료 — talk/total 피드백 NULL 유지, v1.3) |
| `AQ` | NUMBER(3) | nullable, CHECK(0~100) | **세션 총점** — 8문제 점수만으로 산출 (AI 대화 미포함, v1.3 확정). /report/problems 시점에 적재. 리포트 전 NULL |
| `LISTEN_FEEDBACK` 🆕 | CLOB | nullable | 알아듣기 지표 피드백 — /report/problems 시점 적재 |
| `NAMING_FEEDBACK` 🆕 | CLOB | nullable | 이름대기 지표 피드백 — 동일 |
| `SHADOWING_FEEDBACK` 🆕 | CLOB | nullable | 따라말하기 지표 피드백 — 동일 |
| `SELF_TALK_FEEDBACK` 🆕 | CLOB | nullable | 스스로말하기 지표 피드백 — 동일 |
| `TALK_FEEDBACK` 🆕 | CLOB | nullable | AI 대화(STORYTELLING) 피드백 — **/report/total 시점 적재 (2단계 중 2차)** |
| `TOTAL_FEEDBACK` 🆕 | CLOB | nullable | 종합 피드백 — 동일 |
| `REPORT_VIEWED_AT` 🆕 (v1.3) | TIMESTAMP | nullable | 클라가 상세 보고서를 조회한 시점. **null=미조회** — 앱 내 알림함/네비 버블 판별용 (조회 시각 저장으로 조회 여부+시점 동시 커버) |
| `CREATED_AT` / `UPDATED_AT` | TIMESTAMP | NOT NULL | |

> REPORT 테이블은 **신설하지 않음** (ADR-008) — 리포트 내용이 세션 행에 통합됨. 턴별 상세는 TURN 조회로 충분.

### 4.5 TURN (턴 / Q&A 1세트)

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `ID` | NUMBER(19) | PK, TURN_SEQ | |
| `SESSION_ID` | NUMBER(19) | NOT NULL, FK→learning_session | |
| `TURN_NUMBER` | NUMBER(5) | NOT NULL | 세션 내 순서 |
| `CONTENT_TYPE` | VARCHAR2(20) | NOT NULL, FK→content_type, CHECK IN 6종 | |
| `STATUS` 🆕 | VARCHAR2(20) | | **PENDING**(출제·미풀이) / **SUBMITTED**(답안 제출) / **SCORED**(채점 완료) |
| `PROMPT_TEXT` | CLOB | | AI 제시 텍스트 (TTS 지문 / STORYTELLING AI 발화) |
| `CHOICES_JSON` | CLOB | LISTEN_TEXT·LISTEN_PICTURE 필수 | `[{order, media_type: TEXT\|IMAGE, ref}]` — **v2.3: 유형별 고정** (LISTEN_TEXT=TEXT만, LISTEN_PICTURE=IMAGE만 — 혼합 폐지) |
| `CORRECT_VALUE` | VARCHAR2(255) | LISTEN·NAMING 필수 | LISTEN=정답 choice ref / NAMING=정답 단어 / SHADOWING=원문 / SELF_TALK=**NULL** |
| `SELECTED_VALUE` | VARCHAR2(255) | | LISTEN=유저가 탭한 choice ref, 그 외 NULL |
| `ANSWER_TEXT` | CLOB | | 유저 발화 STT (NAMING/SHADOWING/SELF_TALK/STORYTELLING) |
| `HINTS_SHOWN` | NUMBER(1) | | NAMING 힌트 노출 수 (0~2) |
| `SCORE` (v1.95 추가) | NUMBER(5,2) | nullable, 0~100 | **평가지표 채점 결과** — STORYTELLING 항상 NULL |
| `CREATED_AT` | TIMESTAMP | NOT NULL | |

**CHECK 제약:** LISTEN_TEXT·LISTEN_PICTURE → choices_json·correct_value NOT NULL / NAMING → correct_value NOT NULL.

**타입별 컬럼 사용 규격:**

| 타입 | prompt_text | choices_json | correct_value | selected_value | answer_text |
|------|-------------|--------------|---------------|----------------|-------------|
| LISTEN_TEXT | TTS 지문 | 텍스트 선택지 2~4개 (등급별) | 정답 ref | 유저 선택 ref | — (탭 선택) |
| LISTEN_PICTURE | TTS 지문 | 이미지 선택지 2~4개 (등급별) | 정답 ref | 유저 선택 ref | — (탭 선택) |
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
| `THEMA_KEY` | VARCHAR2(30) | CHECK IN('TEST','HOSPITAL','CAFE','TEST_HARD','HOSPITAL_HARD','CAFE_HARD') — **v2.3: HARD 태그 3종 추가** (LISTEN 난이도 운영용) | |
| | | UNIQUE(image_id, thema_key) | 한 이미지가 여러 테마에 속할 수 있음 |

> 세션 테마가 랜덤 결정되면 이 테이블로 해당 테마의 사용 가능한 이미지 풀을 조회 → `/sessions`의 `imageList` 전달. 룩업 테이블 없이 key 문자열 방식 (CONTENT_TYPE 선례).
>
> **v2.3 난이도 운영:** 기존 태그(TEST/HOSPITAL/CAFE) = **EASY** 역할 (사물 이미지) / `_HARD` 태그 3종 = **HARD** (행동이 포함된 상황·사람 이미지). 백엔드가 세션 생성 시 userAQ로 등급 산정(03 계약서 §2.1) → 해당 난이도 태그로 이미지 풀 조회 → imageListListening 필터링. **NAMING은 EASY 이미지 전용** — HARD 이미지는 NAMING 출제 불가.

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
| LISTEN 세분화 (v2.3) | 기존 TURN CONTENT_TYPE=LISTEN 행 → **LISTEN_TEXT UPDATE** (기존 스텁은 텍스트 선택지 반환). CONTENT_TYPE 룩업 seed 갱신 + TURN CHECK 제약 재생성 + IMAGE_THEMA CHECK 확장 필요 |
| USER_REPRESENTATIVE_SCORES 백필 (v2.6) | 기존 세션 보유 계정: USER_AQ·4지표를 기존 LEARNING_SESSION/TURN 집계로 **백필** — 미백필 시 설문 재노출 오검출(null=미응답 판별) |
| 콘텐츠 시딩 | 관리자 페이지로 (이미지 필수 + 태그 JSON 선택) |

## 9. 미구현 (채점 확정 후 대기)

| 항목 | 상태 |
|------|------|
| `TURN.score` 저장 로직 (백엔드) | 계약 구현과 함께 |
| 유저 수준/대시보드 집계 쿼리 | 실력 산정식 확정으로 구현 가능 — **USER_REPRESENTATIVE_SCORES 캐시(v2.6)로 대시보드 조회는 단순 SELECT로 전환** |
| init SQL 05번 → 현행 기준 동기화 | DB 세션 작업 |

## 10. 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v2.0 | 2026-09-02 | **전면 리라이트** — 폐지 테이블/이력 제거, 현행 스키마만 기술. DB 수정 확정사항 반영: IMAGE_RESOURCE cue 2컬럼(IMAGE_HINT_PATH 폐지), LEARNING_SESSION AQ+피드백 6컬럼, VOICE_RECORD rename(SPEAKING_TIME/ARTICULATION_TIME), USER_PROFILE likes/sex/age, TURN.status. OCI hint.json 규약 폐지 |
| v2.3 | 2026-09-04 | **컨테이너 협의 확정 (3) — LISTEN 세분화 + 난이도 태그:** (1) CONTENT_TYPE 세분화 — LISTEN 폐지 → **LISTEN_TEXT/LISTEN_PICTURE** (지표상 동일 LISTEN, 운영 구분용 — 다시보기·난이도). TURN CHECK 5→6종, choices_json 유형별 고정(LISTEN_TEXT=TEXT만/LISTEN_PICTURE=IMAGE만). 기존 LISTEN TURN 행은 LISTEN_TEXT UPDATE. (2) IMAGE_THEMA.THEMA_KEY CHECK 확장 — **HARD 태그 3종 추가**(TEST_HARD/HOSPITAL_HARD/CAFE_HARD; 기존 3종=EASY 역할, 사물 vs 행동·상황·사람). (3) imageListListening = userAQ 등급(03 §2.1) 기반 태그 필터 결과. NAMING=EASY 이미지 전용. LIVE DB 마이그레이션(룩업 seed·TURN CHECK·IMAGE_THEMA CHECK·기존 행 UPDATE)은 후속 DB 작업 |



| v2.6 | 2026-09-04 | **컨테이너 협의 확정 (7) — 대시보드 실구현 설계:** (1) **USER_REPRESENTATIVE_SCORES 신설(§4.2.2)** — USER_PROFILE.USER_AQ(안) 포함 대표점수 5종 이동. LISTEN=TEXT/PICTURE 통합. 갱신 지점 2곳 고정(설문 접수+/report/problems 수신). /sessions userAQ=LEFT JOIN. ⚠️ 기존 계정 백필 필요. (2) LEARNING_SESSION **SESSION_NAME** 신설 — 카드 표시명(today=`오늘의 학습 - {테마명}`, theme=시나리오명). (3) 방사형 출처 구분 — 대시보드=대표점수 / 세부 보고서=해당 세션 TURN.score 집계. (4) 학습 기록·세부 보고서 조회 STATUS != COMPLETED_NO_TALK (학습 중단 미표시) |
| v2.1 | 2026-09-04 | **컨테이너 협의 반영 (1) — 리포트 2단계 + 세션 분기:** LEARNING_SESSION에 `TYPE`(today/theme — 컨테이너 엔드포인트 분기 매핑), `REPORT_VIEWED_AT`(상세 보고서 조회 시각, null=미조회 — 알림/버블 판별) 신설. STATUS에 `COMPLETED_NO_TALK` 추가(이야기 없이 조기종료 — /report/total 미호출, talk/total 피드백 NULL 유지). AQ 산정 시점 확정 = /report/problems(8문제만). 피드백 적재 2단계화(4지표=problems, talk/total=total). LIVE DB 마이그레이션은 후속 DB 작업(init SQL 05번 갱신 포함) | **컨테이너 협의 반영 (1) — 리포트 2단계 + 세션 분기:** LEARNING_SESSION에 `TYPE`(today/theme — 컨테이너 엔드포인트 분기 매핑), `REPORT_VIEWED_AT`(상세 보고서 조회 시각, null=미조회 — 알림/버블 판별) 신설. STATUS에 `COMPLETED_NO_TALK` 추가(이야기 없이 조기종료 — /report/total 미호출, talk/total 피드백 NULL 유지). AQ 산정 시점 확정 = /report/problems(8문제만). 피드백 적재 2단계화(4지표=problems, talk/total=total). LIVE DB 마이그레이션은 후속 DB 작업(init SQL 05번 갱신 포함) |
| (구 v0.5 이하) | ~2026-09-01 | 세션/턴/녹음 재설계 이력은 archive 및 계획서 §15 참고 |