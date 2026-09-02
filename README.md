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
| [04_Database_Design.md](04_Database_Design.md) | DB 설계 — 현행 스키마 전체 (9테이블 + 제약 + 규약) | 중간 |
| [05_API_Design.md](05_API_Design.md) | 클라 ↔ BE API — 인증/사용자/세션/턴/리포트 | 개발 중 |
| [06_Session_Flow_Spec.md](06_Session_Flow_Spec.md) | **세션 기획 단일 진실 원천** — 턴 구조, 타입 매트릭스, 채점/리포트 흐름, pending | 중간 |
| [07_WBS.md](07_WBS.md) | WBS / 개발 일정 / 팀 역할 | 낮음 |
| `adr/` | 아키텍처 의사결정 기록 (ADR-001~010) | 결정 시 |
| `archive/` | 구 문서 (AI 파이프라인 설계, 와이어프레임, 체크리스트 등) | — |

## 현재 진행 상황 요약 (2026-09-02)

| 영역 | 상태 |
|------|------|
| 인증/회원/프로필 (Android + BE E2E) | ✅ |
| Oracle DB + 관리자 페이지 | ✅ |
| 음성 업로드 API + 녹음 클라 + 권한 UX + nginx | ✅ |
| 세션/턴/녹음 DDL | ✅ (수정사항 확정 — 적용 대기) |
| **세션 문제풀이 API (출제·채점·리포트)** | 🔜 이번 단위 |
| AI 컨테이너 VM 배포 + BE 연동 | 🔜 이번 단위 |
| 대시보드/기록/설정 | ⏳ Phase 4 |
| 최종 발표 | 9/10 |

## 문서 규칙

- 버저닝: 시맨틱 (`vMAJOR.MINOR`) — 전면 리라이트는 MAJOR
- 각 문서의 변경 이력 섹션에 기록
- **시크릿(패스워드, IP, 키) 절대 기입 금지**

## 수정 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v2.0.0 | 2026-09-02 | **전면 재구성:** 문제풀이 기획 확정 반영. 신규 03(계약서)·06(세션플로우), 04·05 전면 리라이트, 02 갱신, 01 표면 갱신, ADR 10건 신설. 구 03/와이어프레임/체크리스트 → archive 이관 |
| (구 v1.x) | ~2026-09-01 | 초기 문서 체계 (archive 참고) |