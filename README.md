# documentations

SeSAC TeamProject 공식 문서 — AI 기반 발화 연습 문제풀이 앱

> **역할 분리 원칙:**
> - `documentations/` = **WHAT** — 스펙·계약·설계 (팀 전체 공유, 자주 안 바뀜)
> - `baseworks/` = **WHO/WHEN** — 태스크·진행상황·세션 지시문 (매 작업마다 갱신). 스펙 재서술 금지, 링크만.
> - 상충 시 우선순위: `06_Session_Flow_Spec.md` / `03_AI_Container_Contract.md` (확정 기획) → baseworks 계획서 → 그 외

## 문서 맵

| 문서 | 내용 | 갱신 빈도 |
|------|------|-----------|
| [01_Requirements.md](01_Requirements.md) | 요구사항 (SRS) — 기능, 유저 스토리, 우선순위 | 낮음 |
| [02_Architecture.md](02_Architecture.md) | 시스템 아키텍처 — 구성도, 음성 3단계 저장, 운영 규칙, 확장 로드맵 | 중간 |
| [03_AI_Container_Contract.md](03_AI_Container_Contract.md) | **BE ↔ AI 컨테이너 API 계약** — 엔드포인트, 스키마, 산정식 | 개발 중 |
| [03a_AI_Container_API_Reference.md](03a_AI_Container_API_Reference.md) | **BE↔AI컨테이너 실구현 명세서** (엔드포인트별 JSON 예시, 컨테이너 구현자용) | 개발 중 |
| [04_Database_Design.md](04_Database_Design.md) | DB 설계 — 현행 스키마 전체 (9테이블 + 제약 + 규약) | 중간 |
| [05_API_Design.md](05_API_Design.md) | 클라 ↔ BE API — 인증/사용자/세션/턴/리포트 | 개발 중 |
| [05a_Client_API_Reference.md](05a_Client_API_Reference.md) | **클라↔BE 실구현 명세서** (demo 기준 역추적 + 스텁↔real 전환 가이드) | 개발 중 |
| [06_Session_Flow_Spec.md](06_Session_Flow_Spec.md) | **세션 기획 단일 진실 원천** — 턴 구조, 타입 매트릭스, 채점/리포트 흐름, pending | 중간 |
| [07_WBS.md](07_WBS.md) | WBS / 개발 일정 / 팀 역할 | 낮음 |
| `adr/` | 아키텍처 의사결정 기록 (ADR-001~010) | 결정 시 |
| [03a](03a_AI_Container_API_Reference.md) 내 프롬프트 예시 별첨: `11a_memory_prompt_example.md` | userMemory 관리 LLM 프롬프트 예시 | |
| `archive/` | 구 문서 (AI 파이프라인 설계, 와이어프레임, 체크리스트 등) | — |

## 현재 진행 상황 요약 (2026-09-05)

| 영역 | 상태 |
|------|------|
| 인증/회원/프로필 (Android + BE E2E) | ✅ |
| Oracle DB + 관리자 페이지 | ✅ |
| 음성 업로드 API + 녹음 클라 + 권한 UX + nginx | ✅ |
| **오늘의 학습 데모** (BE 스텁 + Android 전 화면) | ✅ 완료 (2026-09-03 — Android 21커밋 514fdce push, 사용자 E2E 확인) |
| 세션/턴/녹음 DDL | ✅ (데모 마이그레이션 적용 — 2026-09-03) |
| **백엔드 전달 3건** (탈퇴500 FK·userText null·이미지 풀 필터) | ✅ 해결 (P3-30, 2026-09-03 — B-1~B-3 실측 검증 완료) |
| **D-1 DB 마이그레이션 2차** (세션 3컬럼·CONTENT_TYPE 6종·HARD 태그·대표점수 테이블·USER_PROFILE 개편·TAGS 15종) | ✅ 완료 (2026-09-05 — P3-33, DB 레포 f71d2ac push. 04 v2.6 기준 LIVE 반영) |
| **D-2 백엔드 Entity 동기화 + 계약 DTO v1.7** (Entity 6종+신설 3종·DTO 3분할·articulationRate·userRT 0 규약·스텁 LISTEN 세분화) | ✅ 완료 (2026-09-05 — demo 7ba5fff+f344473 push, Hibernate validate+E2E 실측) |
| **D-3 가입 플로우 API 개편 + 설문 접수 API** (PATCH /me 확장·tags/survey/scores 신설·설문 AQ 30/70/90 환산·withdraw FK 8단계) | ✅ 완료 (2026-09-05 — demo ed75d76+b604397 push, E2E+탈퇴 풀사이클 실측, 05a v1.5) |
| **컨테이너 협의 반영** (세션 분기·imageList 3분할·리포트 2단계) | 🔜 D-5 (계약 v1.2 반영 완료 — 구현 대기) |
| **userMemory 개인화** (에이전트 메모리 패턴) | 🔜 D-4 (계약 v1.3·DB v2.2 반영 완료 — 구현 대기. Entity/DTO 필드는 D-2에서 선반영 완료) |
| **최종 기획 확정** (LISTEN 세분화·가입 설문·세션 흐름 시간 통제·대시보드 실구현 설계) | 📋 기획 완료 (2026-09-04 협의 최종 — 계약 v1.7·DB v2.6 반영, 구현 대기) |
| 데모 (오늘의 학습) | 🎉 완료 (2026-09-03) — 최종 발표 9/10 |
| 대시보드/기록/설정 | ⏳ D-7 |
| 최종 발표 | 9/10 |

## 문서 규칙

- 버저닝: 시맨틱 (`vMAJOR.MINOR`) — 전면 리라이트는 MAJOR
- 각 문서의 변경 이력 섹션에 기록
- **시크릿(패스워드, IP, 키) 절대 기입 금지**

## 수정 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v2.0.0 | 2026-09-02 | **전면 재구성:** 문제풀이 기획 확정 반영. 신규 03(계약서)·06(세션플로우), 04·05 전면 리라이트, 02 갱신, 01 표면 갱신, ADR 10건 신설. 구 03/와이어프레임/체크리스트 → archive 이관 |
| v2.0.1 | 2026-09-03 | 데모 백엔드 구현 완료 반영 (05 §7 추가, 진행 요약 갱신) — demo 브랜치 스텁 플로우, DB 데모 마이그레이션 적용 |
| v2.0.2 | 2026-09-03 | Android 데모 화면 전체 완료 반영 (P3-26~29, 21커밋 514fdce push) — 스플래시/프로필수정/문제풀이4타입/이야기3턴/리포트방사형/UI재편(네비 4탭 재편·History 폐지), TokenAuthenticator(만료 실측 403), 진행 요약 갱신. 백엔드 전달 3건은 후속 세션 대기 |
| v2.0.3 | 2026-09-03 | 백엔드 보완 완료 반영 (P3-30, demo 브랜치 44fe801..0dfda5a push) — 회원탈퇴 FK 역순 하드딜리트(500→204, DB 잔존 0 실측)·스텁 userText 더미 STT·출제 이미지 풀 타입별 필터(namingImageIds/selfTalkImageIds 계약 선택 필드 추가). 진행 요약 갱신 |
| v2.0.7 | 2026-09-05 | D-1 DB 마이그레이션 2차 완료 반영 (P3-33, DB 레포 f71d2ac) — LEARNING_SESSION type/session_name/report_viewed_at, CONTENT_TYPE 6종(LISTEN 세분화), IMAGE_THEMA HARD 3종, USER_REPRESENTATIVE_SCORES 신설+백필, USER_PROFILE 개편(hobbies/birth_date/user_memory)+TAGS 15종, init SQL v2.0. 진행 요약 갱신 — 잔여 구현은 D-2~D-8 계획으로 트래킹 |
| v2.0.8 | 2026-09-05 | D-2 백엔드 Entity 동기화 + 계약 DTO v1.7 완료 반영 (demo 7ba5fff+f344473 push) — Entity: UserProfile(hobbies/birthDate/userMemory), Tag·UserProfileTag·UserRepresentativeScore 신설, Session(type/sessionName/reportViewedAt), Turn 6종 정정. DTO: ContainerUserInfo 개편, CreateSessionRequest 3분할, ShadowingScoreRequest.articulationRate, userRT 0 규약. 스텁 listenText/listenPicture 반환. Hibernate validate+E2E 실측(세션 62, TURN 6종 적재, LISTEN 100 채점). 진행 요약 갱신 |
| v2.0.9 | 2026-09-05 | D-3 가입 플로우 API 개편 + 설문 접수 API 완료 반영 (demo ed75d76+b604397 push) — PATCH /me 확장(hobbies/sex/birthDate/tagIds 전량 교체), GET /me/tags(15종), POST /me/survey(서버 산출 AQ 30/70/90 + REP_SCORES upsert), GET /me/scores(대표점수), withdraw FK 역순 8단계(UPS·REP_SCORES 추가), UserDto 확장 5필드, NoSuchElement→E0404 핸들러. E2E 실측(설문 30→73 복구·scores 백필값 일치·임시계정 탈퇴 204 잔존 0) — 05a v1.5 갱신. 진행 요약 갱신 |
| v2.0.4 | 2026-09-04 | **컨테이너 협의 반영 (1):** 03a v1.2(/sessions/today·theme 분기, imageList 3분할, /report/problems·total 2단계), 03 v1.2, 04 v2.1(TYPE·REPORT_VIEWED_AT·COMPLETED_NO_TALK), 06 v1.1(리포트 2단계 UX), 계획서 v2.02(P3-31~33). 데모 완료 표기 |

| v2.0.6 | 2026-09-04 | **컨테이너 협의 확정 (3~7) — 기획 최종본:** 03 v1.7(articulationRate·첫사용 0 전송·LISTEN 세분화 listenText/listenPicture·등급표·중단/완료 판정·대표점수 테이블), 03a v1.7(동일 + JSON 예시 갱신), 04 v2.6(CONTENT_TYPE 6종·IMAGE_THEMA HARD 3종·USER_REPRESENTATIVE_SCORES 신설·LEARNING_SESSION.SESSION_NAME), 05a v1.4(§3.1a 클라 흐름 규약·§8 대시보드/세부보고서 API 3종), 06 v1.7(시간 통제 30초 통일·문제 가이드 화면·중단/완료 판정·가입 설문 §5.2·대시보드 §7.1). 마스코트명 "덕분이" 확정. 기획 확정 후 잔여: 컨테이너 연결·polishing·버그수정·자원관리 |
| (구 v1.x) | ~2026-09-01 | 초기 문서 체계 (archive 참고) |