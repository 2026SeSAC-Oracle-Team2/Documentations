# documentations

SeSAC TeamProject의 문서 정리

## 프로젝트 개요

AI 기반 발화 연습 앱 — Android 클라이언트 + Spring Boot 백엔드 + AI 컨테이너(LLM/채점) + Oracle XE DB

### 리포지토리 구조

| 리포지토리 | 기술 스택 | 설명 |
|-----------|----------|------|
| `SeSAC_SpeechApp_Backend` | Kotlin / Spring Boot 3.x | 모놀리틱 API 서버 |
| `SeSAC_SpeechApp_Frontend` | Kotlin (Android) | Android 클라이언트 앱 |
| `SeSAC_db_container` | Oracle XE (Docker) | 데이터베이스 컨테이너 정의 |
| `SeSAC_llm_container` | Python / FastAPI | LLM 컨테이너 (STT + LLM + TTS) |
| `SeSAC_scoring_container` | Python / FastAPI | 발화 채점 컨테이너 |
| `SeSAC_deployment` | Docker Compose | 전체 서비스 오케스트레이션 |

### 현재 진행 상황

| 항목 | 상태 | 비고 |
|------|------|------|
| **Firebase Auth + JWT** | ✅ 완료 | Backend 구현 완료 |
| **Auth API** (3개 엔드포인트) | ✅ 완료 | POST /auth/firebase, /auth/refresh, /auth/logout |
| **User API** (부분) | ✅ 완료 | GET /users/me, PATCH /users/me |
| **Oracle DB 전환** | ✅ 완료 | BE-DB-01~04 해결, Oracle XE 연동 (2026-08-30) |
| **회원탈퇴 API** | ✅ 완료 | DELETE /users/me (DB hard delete + Firebase afterCommit) (2026-08-30) |
| **프로필 사진 API** | ✅ 완료 | POST/GET /users/me/profile-image (OCI Object Storage 연동) (2026-08-30) |
| **OCI Object Storage SDK** | ✅ 완료 | ObjectStorageService 생성, 프로필 사진 업로드/다운로드 검증 (2026-08-30) |
| **회원가입 플로우** | ✅ 완료 | SignUpActivity 신규, 닉네임 필수, 프로필 사진 선택 (2026-08-30) |
| **프로필 실데이터 표시** | ✅ 완료 | Coil 이미지 로딩 + placeholder (2026-08-30) |
| **End-to-end 검증** | ✅ 완료 | 구글 로그인 → 회원가입 → 프로필 사진 표시 (2026-08-30) |
| **Android Google Sign-In** | ✅ 완료 | 클라이언트 연동 완료 |
| **UI/UX 디자인 시스템** | ✅ 완료 | Trust Blue 테마 |
| **Infra: Firebase** | ✅ 완료 | 프로젝트 설정 완료 |
| **Infra: OCI** | ✅ 완료 | Object Storage 설정 완료 |
| **Infra: OpenAI** | ✅ 완료 | API Key 설정 완료 |
| **Infra: Docker** | ✅ 완료 | Oracle DB 컨테이너 구동 중 |
| **DB 스키마 분리** | ✅ 완료 | SPEECHAPP_USER / SPEECHAPP_CONTENT 분리 |
| **이미지 리소스 테이블** | ✅ 완료 (v1.6) | IMAGE_RESOURCE 단일 테이블 — 태그/힌트는 OCI JSON 파일화로 TAG/HINT 테이블 폐지 |
| **세션/턴/녹음 DB 재설계** | ✅ 설계 확정 (2026-08-31) | 기획 전환: 문제풀이 앱. CONTENT_TYPE·SESSION·TURN·TURN_IMAGE·VOICE_RECORD 5종 — ERD 검증 통과(ERD_session_redesign_draft.md). 실구현(DDL/마이그레이션/API)은 후속 |
| **음성 업로드 API (P3-19)** | 📋 재설계 완료 | POST /api/v1/voice/upload (multipart 50MB). OCI 규약 containers/llm/{uuid}/{session}/{turn}_user.m4a. nginx A안(백엔드 Docker화) 병행 |
| **녹음 권한 UX (P3-22)** | 📋 확정 | 온보딩 안내 화면(스플래시 후·로그인 전·최초 1회), 거부 시 재노출. MediaRecorder AAC/m4a |
| **컨텐츠 리소스 API** | 📋 설계 완료 | 구현 예정 |
| **WebSocket AI 대화** | 📋 설계 완료 | 구현 예정 |
| **LLM 컨테이너** | 📋 설계 완료 | 구현 예정 |
| **채점 컨테이너** | 📋 설계 완료 | 내부 구현 TBD |

## 문서별 설명

| 문서명 | 설명 |
|---|---|
| 01_Software_Requirements_Specification.md | 기능 요구사항, 사용자 스토리, Use Case, P0~P5 우선순위 정의 |
| 02_System_Design_Document.md | 시스템 아키텍처, 리포지토리 분리 구조, Docker Compose 구성도, Spring Boot + Oracle XE + nginx 구조, 배포 구조(SeSAC_deployment) |
| 03_AI_Pipeline_Design.md | 음성 대화 흐름 상세: LLM 컨테이너(Whisper STT + GPT-4o-mini + TTS) → 채점 컨테이너, 별도 리포지토리 구조 반영 |
| 04_Database_Design.md | Oracle 21c XE 기반 관계형 스키마, ERD, DDL, JPA 매핑 규칙, 스키마 분리(SPEECHAPP_USER/CONTENT). **v0.5: 세션/턴/녹음 전면 재설계** — CONTENT_TYPE·SESSION·TURN·TURN_IMAGE·VOICE_RECORD |
| 05_API_Interface_Design.md | REST API + WebSocket API 규격, Firebase Auth 인증, 요청/응답 포맷, 구현 상태 표시, 컨텐츠 리소스 API |
| 06_wireframe_plain.html | 와이어프레임 HTML (P0~P3 화면, 4개 Bottom Tab) |
| 07_WBS_Development_Plan.md | Sprint 기반 개발 일정, 체크포인트(8/25, 8/28, 9/1, 9/4), 간트차트, 태스크 분해 |
| checklist/ | 기능별 개발 체크리스트 및 진행 상황 관리 |
| service_flow/ | 서비스 흐름도 관련 자료 |
| wireframes/ | 화면별 와이어프레임 이미지/파일 |

## 버전 관리

시맨틱 버저닝(Semantic Versioning) 적용: `vMAJOR.MINOR.PATCH`

- **MAJOR**: 문서 전체 재편성, 프로젝트 스코프 변경, 큰 구조 개편
- **MINOR**: 새로운 문서 추가, 기존 문서의 기능/범위 변동
- **PATCH**: 오타 수정, 문서 내용 보완, 포맷 변경, 자잘한 업데이트

## 수정 이력

| 버전 | 수정 대상 | 수정 내용 |
|--|--|--|
| v1.0.0 | - | 각 문서 첫 생성 (1~7) |
| v1.1.0 | 01~07 | 회의 결정 반영: 컨테이너 구조 변경(LLM 통합+채점), Firebase Auth, 컨텐츠 3개(A/B/C), 5개 평가지표, 유저 수준 시스템, 필수/민감정보 분리, 녹음 30초 제한, 이전 학습 기록, 발표일 9/4→9/10 |
| v1.1.1 | 07 | WBS 체크포인트 4개 추가(8/25, 8/28, 9/1, 9/4), 병렬 작업(컨텐츠조사/모델조사/인프라세팅) 반영, 의존성 완화 |
| v1.2.0 | 02,03,04,05,README | 리포지토리 분리 구조 반영(각 컨테이너별 별도 GitHub 리포지토리, SeSAC_deployment 오케스트레이션 추가). DB 스키마 분리(SPEECHAPP_USER/CONTENT) 및 DB 사용자 계정 정의. IMAGE_RESOURCE, IMAGE_TAG, IMAGE_HINT 테이블 신규 추가. Object Storage 2버킷 분리 및 메타데이터 저장 원칙 명시. 인증 API/User API 구현 완료 표시. 컨텐츠 리소스 API(이미지 조회/등록/수정/삭제) 신규 추가. 현재 진행 상황 테이블 추가 |
| v1.4.0 | README, 05 | **"DB 기반 로그인 구현" 완료 반영 (2026-08-30).** 진행 상황 테이블 신규 완료 항목 7개 추가: Oracle DB 전환, 회원탈퇴 API, 프로필 사진 API, OCI Object Storage SDK, 회원가입 플로우, 프로필 실데이터 표시, End-to-end 검증. 음성 업로드 API(ObjectStorageService 재사용 가능) 행 추가. API 인터페이스 설계서(05)에 회원탈퇴(DELETE /users/me), 프로필 이미지 조회(GET /users/me/profile-image) 엔드포인트 신규 추가 및 구현 완료 표시. JSON 네이밍 컨벤션(응답 camelCase, 요청 DTO 일부 snake_case) 노트 추가. |
