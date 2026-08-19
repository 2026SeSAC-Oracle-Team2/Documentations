# documentations

SeSAC TeamProject의 문서 정리

## 문서별 설명

| 문서명 | 설명 |
|---|---|
| 01_Software_Requirements_Specification.md | 기능 요구사항, 사용자 스토리, Use Case, P0~P5 우선순위 정의 |
| 02_System_Design_Document.md | 시스템 아키텍처, Docker Compose 구성도, Spring Boot + Oracle XE + nginx 구조 |
| 03_AI_Pipeline_Design.md | 음성 대화 흐름 상세: LLM 컨테이너(Whisper STT + GPT-4o-mini + TTS) → 채점 컨테이너 |
| 04_Database_Design.md | Oracle 21c XE 기반 관계형 스키마, ERD, DDL, JPA 매핑 규칙 |
| 05_API_Interface_Design.md | REST API + WebSocket API 규격, Firebase Auth 인증, 요청/응답 포맷 |
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
