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
| [diagrams/architecture.md](diagrams/architecture.md) | **시스템 아키텍처 Mermaid 구조도** (02 v2.1 기준 — 발표용) | 중간 |
| [03_AI_Container_Contract.md](03_AI_Container_Contract.md) | **BE ↔ AI 컨테이너 API 계약** — 엔드포인트, 스키마, 산정식 | 개발 중 |
| [03a_AI_Container_API_Reference.md](03a_AI_Container_API_Reference.md) | **BE↔AI컨테이너 실구현 명세서** (엔드포인트별 JSON 예시, 컨테이너 구현자용 — v1.12 TTS 스킵·LISTEN 폴백 한정·톤 규약 구현 확정·STT 재시도/503) | 개발 중 |
| [04_Database_Design.md](04_Database_Design.md) | DB 설계 — 현행 스키마 전체 (9테이블 + 제약 + 규약) | 중간 |
| [05_API_Design.md](05_API_Design.md) | 클라 ↔ BE API — 인증/사용자/세션/턴/리포트 | 개발 중 |
| [05a_Client_API_Reference.md](05a_Client_API_Reference.md) | **클라↔BE 실구현 명세서** (demo 기준 역추적 + 스텁↔real 전환 가이드 — v1.10 음성 제출 비동기 재편·talk 첫 턴 소비 시점 계약) | 개발 중 |
| [06_Session_Flow_Spec.md](06_Session_Flow_Spec.md) | **세션 기획 단일 진실 원천** — 턴 구조, 타입 매트릭스, 채점/리포트 흐름, pending (v1.9 — e2e-3 보고서 화면 분리 반영) | 중간 |
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
| **컨테이너 협의 반영** (세션 분기·imageList 3분할·리포트 2단계) | ✅ 완료 (2026-09-06 — D-5, demo e3f104e+38a80ac+9521f2b push: /sessions/today·theme 분기, 리포트 2단계 afterCommit 백그라운드, 중단/완료 판정, history/report API, userAQ 캐시 교체, articulationRate 최근 20 창. 03a v1.9·05a v1.6) |
| **D-6 가입 플로우 UI 개편 + 가입 설문** (성별·생년월일·취미·태그 버블 단계 + 5문항 설문 + userAq 재노출 게이트) | ✅ 완료 (2026-09-06 — Android demo 566c087+b3eee57 push. 가입 3단계 플로우·설문 스킵 불가·서버 산출 userAq 안내. ProfileEdit 확장은 D-7 이월) |
| **D-7 학습 화면 시간 통제 + AI 대화 UX + 대시보드/세부 보고서** (06 §3 시간통제 전면·ProblemGuideActivity·세션 today/theme 연동·Storytelling 중단/마치기·8턴 하드캡·대시보드 실데이터화·SessionDetailActivity) | ✅ 완료 (2026-09-06 — Android demo ee2c41a+1de328e+ed4bc42 push, ls-remote 일치. WSL 실빌드 3회 SUCCESSFUL. ProfileEdit 확장·사용자 에뮬레이터 E2E는 D-7b 이월) |
| **userMemory 개인화** (에이전트 메모리 패턴) | ✅ 완료 (2026-09-05 — D-4, demo 0e8221f+dbc480e+2db0b53 push: /report/total 갱신 라이프사이클+하드캡 8192문자·userInfos.tags 주입 완성·세션 type/session_name 적재·스텁 갱신 시뮬레이션. 03a v1.8) |
| **최종 기획 확정** (LISTEN 세분화·가입 설문·세션 흐름 시간 통제·대시보드 실구현 설계) | 📋 기획 완료 (2026-09-04 협의 최종 — 계약 v1.7·DB v2.6 반영, 구현 대기) |
| 데모 (오늘의 학습) | 🎉 완료 (2026-09-03) — 최종 발표 9/10 |
| 대시보드/기록/설정 | ✅ D-7 (2026-09-06 — Android demo ee2c41a+1de328e+ed4bc42 push. 대시보드 실데이터화+세부 보고서 SessionDetailActivity. ProfileEdit 확장·E2E는 D-7b 이월) |
| **데모 UX 라운드** (F-5 유형 매트릭스 + F-7 수정1~7 + F-8 로딩/가이드/AI대화 연결) | ✅ 완료 (2026-09-08 — Android feat/f6-session-ux 31커밋 main 머지 e809df7. 유형×요소 기획 정합·오버레이 제출·카운트다운 통일·가이드 4단+예시 리소스·AI 대화 talk 연결 — 사용자 실기 전항목 통과) |
| F-6 데모 리소스·디버그 도구 | ⏸️ 미실시 (사용자 판단 — 데모 UX 직접 테스트로 대체) |
| v2.2.2 | 2026-09-09 | **아키텍처 구조도 신설 + 02 v2.1 (사용자 확정분 — VM 실물 검증 완료):** diagrams/architecture.md 신설(발표용 Mermaid), 02 v2.1 — AI 모델 스택 실측 정정(LLM=Ollama Cloud·STT=Whisper large-v3-turbo+LoRA lora_v2·TTS=로컬 Qwen→**OpenAI TTS API** 고정 프리셋) + §3 구성도 전면 갱신(공유폴더 음성 교환·DB=OCI 블록 볼륨 bind mount /mnt/db_data(/dev/sdb)·관리자 페이지) + §2 ai_container VM 실가동 표기. 검증: Deployment compose volumes(/mnt/db_data:/opt/oracle/oradata)·AI컨테이너 main.py/hf_stt.py/config.py 실물 코드 확인 |
| v2.2.1 | 2026-09-09 | **e2e-3 확정분 + srv-2 문서 반영 (문서화 세션):** 05a v1.9(§3.1 demo.themes TEST→HOSPITAL,CAFE·예시 정합화), 06 v1.9(보고서 화면 분리 — 간이=AI대화 미포함·상세=기록탭 전용), 03a v1.11(§0 shared-audio-root·TTS 실물 스트리밍 기술 보강 + §6.1 AI 발화 톤 규약 2~3문장·이모티콘/반복자음 금지). 보류(구현 완료 게이트): 음성 제출 비동기(e2e-3 A)·POST /sessions/chat(F)·SELF_TALK tags.json 원본 연동 표기. 커밋 4da6525·3130a4c·78be01d — push 대기 |
| v2.2.0 | 2026-09-08 | **데모 UX 라운드 완료 반영 (FE feat/f6-session-ux 31커밋 main 머지 e809df7, ls-remote 일치·Author NonokEE <shshrdl@naver.com> 통일):** F-5 유형×요소 매트릭스 기획 정합(LISTEN_PICTURE 이미지 그리드·WAIT 사진 관찰·가이드 이미지 제거·턴 잔존 차단 — 4커밋), F-7 수정1~7(힌트 카드 누적·낭독TTS 폐지 A2·다시듣기 화면유지+카운트다운 연속·선택지 하이라이트 라운드클립·카운트다운 통일 6커밋·카드 위치/크기 통일·제출 오버레이 fade-in+30초 강제제출 전용 문구 — 17커밋), F-8(로딩 화면 시안 L2 bounce+첫 프레임 렌더·가이드 4단+예시 리소스 4종 nodpi·AI 대화 연결 — initChat 후 talk multipart·@Multipart+최소 1 part 2회 정정·B-10/B-11 회귀 정정 — 7커밋), B-10 상단 480dp 공백+B-11 말풍선 곡률 반전 해소. BE 무접촉(curl 프로브만)·finish 호출 시점 계약 확정(AI대화 종료 1회 — 8턴 직후 호출 시 COMPLETED 선점으로 talk E0401 실측). 검증: WSL 실빌드 15회+ SUCCESSFUL·사용자 실기 전항목 통과·APK 21종+md5 전수 기록. 문서: 06 v1.8·05a v1.8·03a v1.10 갱신. F-6 데모리소스·디버그 도구 미실시(사용자 판단) |
| v2.1.6 | 2026-09-06 |

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
| v2.1.1 | 2026-09-06 | D-5 세션 2종 분기+리포트 2단계+대시보드 API 실구현 완료 반영 (demo e3f104e+38a80ac+9521f2b push, ls-remote 일치) — /sessions/today·/theme 2종 분기(thema 유효성 E0400·/v2 하위호환), 리포트 2단계 재편(8문제 채점 감지 afterCommit 백그라운드 /report/problems + finish 백그라운드 /report/total — REQUIRES_NEW 적용자 분리), 중단/완료 판정(유저 talk 답변 수 기준 — 1~3턴 COMPLETED_NO_TALK total 미호출 / 4턴 이상 COMPLETED), talk-turn-limit 3→8(유저 답변 수 기준 하드캡), REPORT_VIEWED_AT(null일 때만 기록), userAQ=REP_SCORES.USER_AQ 캐시 교체, articulationRate/userRT 최근 20개 창 수정, 대시보드 API 3종 실구현(05a §8.2 AQ NOT NULL 규약 확정·§8.3 radar/metricCards/talkHistory/answer 계약), B-4 스텁 TTS 매핑 확정. 검증 ①~⑨ 실측 — articulationRate 11.64 수동 대조 일치·AQ 손계산(81/75) 일치·탈퇴 풀사이클(임시계정 id 46→204 잔존 0). 문서: 03a v1.9 + 05a v1.6 |
| v2.1.5 | 2026-09-06 | D-8②b 기획 누락분 소화(홈 통계 API + 홈 카드 실데이터) 완료 반영 (BE demo 9b35395 / FE demo 1fc9825 push, ls-remote 일치) — BE: **GET /api/v1/users/me/stats 신설**(JWT·scores 동일 봉투·05a §8.4) — streakDays(완료 세션 일자 연속·중단 세션 제외·오늘/어제 소급 — 오늘 완료 없어도 어제까지 연속이면 유지)·avgScore(최근 10개 완료 세션 AQ 평균 소수 1자리 — **ADR-009 아님, 최근 10개 전부 평균**)·deltaScore(최근 10 평균 − 직전 10 평균·11번째 없으면 null). 공용 프레디케이트 CompletedSessionFilter(history·stats 공유 — COMPLETED_NO_TALK에 간이 AQ 적재 케이스 실측, 단일 aq!=null 필터 오염 방지). FE: 홈 탭(LearnFragment) 통계 2카드 실데이터(연속 학습 N일/평균 점수 +Δ — onResume 재조회·조회 실패 시 "-" 폴백·delta null 증감 숨김·부호 포맷 클라 담당), 오늘의 학습 배지 문구 교체("문제 8개 + 덕분이 대화, 약 8분" — strings_d8b.xml 신설, strings.xml 커밋 금지 유지), 테마별 화면 DuckSays 배너(텍스트만·마스코트 에셋은 D-8③). 검증: user 26 실측 streak=1·avg=72.3(sqlplus 손계산 일치)·임시계정 4건 경계 실측(신규 0/null·delta null·중단세션 불반영·AQ null 불반영·4턴 완료 streak=1) 후 탈퇴 204 잔존 0·회귀 scores/history/report76 diff 0·compileKotlin+WSL 실빌드 SUCCESSFUL·kt_check+Binding/R.string/DTO↔JSON 대조 OK |
| v2.1.6 | 2026-09-06 | D-8③ 디자인 시안 실구현 완료(~/design React 시안 → Android) — FE feat/d8c-design-apply 4커밋(6a9c660 토큰+마스코트 / 7941840 학습플로우 / c6f56f2 홈·기록·학습탭 / 21de8c3 가입·설문·프로필·로그인·네비), WSL 빌드 4회 SUCCESSFUL·Binding 전수 일치·push ls-remote 일치 — oklch→hex 토큰 변환(명세 앵커 검증)·duck WebP 944KB→21KB·홈 레이더 stub→최근 학습 결과 3개 리스트 대체(사용자 확정)·테마 카드 OCI 이미지 로딩·DuckSays 배너 마스코트 — SSOT 문구 우선(설문/척도/12문항/나가기 팝업 시안 문구 미채택) — demo 병합은 PR 대기·에뮬레이터 시각 검증 사용자 몫 |
| v2.1.4 | 2026-09-06 | D-8② 백엔드 리팩토링(동작 불변) + D-7b ProfileEdit 확장 완료 반영 (BE demo 42c6a49+3a8772f / FE demo d465298 push, ls-remote 일치) — BE: SessionFlowService 1,046행 관심사별 분할(SessionCreationService·SessionScoringService·SessionReportQueryService·ScoreCalculationService·SessionTurnSupport — 퍼사드 유지로 컨트롤러 시그니처 무변경, @Transactional·afterCommit/REQUIRES_NEW 구조 보존), RealAiContainerClient base-url ai.container.base-url 프로퍼티 주입(기본값 유지 — D-8⑤ 준비), 유령 메서드(SessionService.getNextTurnNumber) 제거. FE D-7b: ProfileEdit 확장(성별 Chip·생년월일 DatePicker·취미·태그 15종 역매핑 최대 5개 — PATCH 일괄 부분 업데이트), 레이아웃 NestedScrollView+시니어 UI, 문자열 strings_survey.xml 분리. 검증: **회귀 실측 ①~⑧ 전면 — 리팩토링 전/후 scores·history·report76 JSON diff 0**(동작 불변 증명), REP_SCORES 갱신 손계산 일치, 중단/완료 판정 실측, 탈퇴 풀사이클 204 잔존 0, WSL 실빌드 2회 SUCCESSFUL, kt_check+ID/string 대조 OK. 계약 변화 없음 — 05a 갱신 없음(v1.6 유지) |
| v2.1.3 | 2026-09-06 | D-7 Android 학습 화면 시간 통제 + AI 대화 UX + 대시보드/세부 보고서 실구현 완료 반영 (Android demo ee2c41a+1de328e+ed4bc42 push, ls-remote 일치) — 시간 통제(ProblemGuideActivity 신설 턴별 가이드 06 §3 기획 문구 strings_d7.xml 분리·LoadingActivity [시작] 게이트 마이크 권한 선제 확인·ProblemActivity 대기 카운트다운 3초/5초+제출 30초 시각 표시+LISTEN 30초 도달 CASE2 최근 선택/CASE3 미선택 오답 sentinel 제출+음성형 녹음 컷 강제제출+SHADOWING 다시듣기 제거+제출/다음 버튼 분리+시니어 텍스트 버튼), 세션 2종 분기 연동(createSessionToday/createSessionTheme — /v2 하위호환·LearnFragment today·PracticeFragment CAFE/HOSPITAL), AI 대화 UX([학습 중단하기] 1~3턴 우는 덕분이 팝업→finish 중단 판정·4턴째 [학습 마치기] 전환·8턴 하드캡+마무리 응답 후 결과 보기·"덕분이가 답변을 생각중이에요" 버블·녹음 30초 제한 자동 제출), 대시보드/세부 보고서(DTO 3종 05a §8.1~8.3 키 1:1·ApiService/Repository 3종·DashboardFragment 실데이터화 stub 제거+onResume 재조회+빈 상태·SessionDetailActivity 신설 지표 카드 확장 문제 기록+AI TTS/유저 음성 재생+REPORT_VIEWED_AT 서버 기록). 검증: WSL 실빌드 3회 BUILD SUCCESSFUL + kt_check 전 대상 OK + strings/ID 전수 대조 + DTO↔05a 1:1 9그룹 + 로직 리뷰(타이머 해제·30초 경로·CASE2/3·4턴 카운트·ISO 폴백). 미완료: ProfileEdit 확장·사용자 에뮬레이터 E2E(D-7b 이월) |
| v2.2.2 | 2026-09-09 | **e2e-3 A·F·G·H·C·def 재실측 완료분 반영 (매니저):** 05a v1.10(§3.2 음성 제출 비동기 재편 — VoiceSubmitData 즉시 응답 score=0·TURN SUBMITTED 전환·백그라운드 채점·8턴 감지 완화+멱등 가드 / §3.4 talk 첫 턴 소비 시점 계약 신설 — initChat null 전달 재발 해소 c797070), 03a v1.12(§2.2 LISTEN 폴백 TAG없는 이미지 한정 규약·TTS 스킵 규약 NAMING·SELF_TALK=""·§6.1 톤 규약 구현 확정+클램프 코드 방어·STT 재시도/503 STT_TRANSCRIBE_FAILED). 구현 완료 확인분만 반영 — F(FAB /sessions/chat)·E(튕김)·재생 버튼 최종 수정은 진행 중이라 미반영. 커밋 3130a4c·78be01d·63921b4(문서화 세션) + 본 라운드 |
| v2.1.2 | 2026-09-06 | D-6 Android 가입 플로우 UI 개편 + 가입 설명 화면 완료 반영 (Android demo 566c087+b3eee57 push, ls-remote 일치) — 데이터 계층(UpdateProfileRequest 확장 hobbies/sex/birthDate/tagIds·UserDto 확장 5필드 하위호환·SurveyDtos 신설·ApiService getTags/submitSurvey 2종·UserRepository 3종), 가입 3단계 플로우(Step1 닉네임+사진 → Step2 SignUpProfileStepActivity 성별 Chip·생년월일 DatePicker maxDate=현재·취미·태그 ChipGroup 최대 5개 → 설문 → 메인), SurveyActivity(5문항 06 §5.2 전문 strings_survey.xml 앱 고정·스킵 불가 뒤로가기 경고·전 문항 응답 시 제출 활성화·서버 산출 userAq "학습 준비 완료!" 안내·재응답 항상 제출), 로그인 userAq null 재노출 게이트(isNewUser와 독립 — 기존 isNewUser 로직 무변경). 검증: WSL 실빌드 BUILD SUCCESSFUL + kt_check.py 11파일 OK + Binding↔레이아웃 ID 전수 대조 + R.string 전수 대조 + DTO↔05a v1.6 §2 1:1. strings.xml 더미 커밋 제외(WSL 게이트 준수). 미완료: 사용자 에뮬레이터 E2E(가입 3단계→설문→메인), ProfileEdit 확장은 D-7 이월 |
| v2.1.0 | 2026-09-05 | D-4 userMemory 개인화 구현 완료 반영 (demo 0e8221f+dbc480e+2db0b53 push) — /report/total userMemory 라이프사이클(Report DTO 양방향 필드·finish ①기존값조회→②호출→③갱신규약: null=기존유지 소실방지·8192문자 하드캡 절단·동일트랜잭션 UPDATE→④AQ+피드백 6컬럼 유지), userInfos.tags 주입 완성(UserService.buildTagsString 헬퍼 추출·createSession/talk 적용), 세션 type/session_name 적재, 스텁 userMemory 갱신 반환(기존+더미/신규 작성). 실측: 태그 조립 "건강관리, 등산, 독서"·3케이스(87→126자·null 유지)·하드캡 9000→8192·임시계정 탈퇴 204 잔존 0 — 03a v1.8 갱신. 진행 요약 갱신 |
| v2.0.4 | 2026-09-04 | **컨테이너 협의 반영 (1):** 03a v1.2(/sessions/today·theme 분기, imageList 3분할, /report/problems·total 2단계), 03 v1.2, 04 v2.1(TYPE·REPORT_VIEWED_AT·COMPLETED_NO_TALK), 06 v1.1(리포트 2단계 UX), 계획서 v2.02(P3-31~33). 데모 완료 표기 |

| v2.0.6 | 2026-09-04 | **컨테이너 협의 확정 (3~7) — 기획 최종본:** 03 v1.7(articulationRate·첫사용 0 전송·LISTEN 세분화 listenText/listenPicture·등급표·중단/완료 판정·대표점수 테이블), 03a v1.7(동일 + JSON 예시 갱신), 04 v2.6(CONTENT_TYPE 6종·IMAGE_THEMA HARD 3종·USER_REPRESENTATIVE_SCORES 신설·LEARNING_SESSION.SESSION_NAME), 05a v1.4(§3.1a 클라 흐름 규약·§8 대시보드/세부보고서 API 3종), 06 v1.7(시간 통제 30초 통일·문제 가이드 화면·중단/완료 판정·가입 설문 §5.2·대시보드 §7.1). 마스코트명 "덕분이" 확정. 기획 확정 후 잔여: 컨테이너 연결·polishing·버그수정·자원관리 |
| (구 v1.x) | ~2026-09-01 | 초기 문서 체계 (archive 참고) |
