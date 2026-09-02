# 세션 플로우 스펙 (Session Flow Specification)

> **버전:** v1.0 (2026-09-02 기획 확정)
> **역할:** 문제풀이 세션 기획의 **단일 진실 원천(Single Source of Truth)**. 화면 흐름·상태·채점·리포트의 기획 확정본.
> **관련 문서:** API 계약(BE↔AI컨테이너) = `03_AI_Container_Contract.md` · 클라↔BE API = `05_API_Design.md` · 스키마 = `04_Database_Design.md`

---

## 1. 용어 정의 (팀 공식)

| 용어 | 정의 |
|------|------|
| **AI 컨테이너** | 통합 AI 컨테이너 — 문제 출제 + 채점 + 리포트 담당 (구 llm-container + scoring-container 통합) |
| **발화지표** | `VOICE_RECORD`의 `SYLLABLES` / `SPEAKING_TIME` / `ARTICULATION_TIME` 3종. AI 컨테이너가 측정 → 백엔드 적재 |
| **평가지표** | LISTEN·NAMING·SHADOWING·SELF_TALK (STORYTELLING 제외) — 턴 타입 = 채점 지표. 결과 점수는 `TURN.score`에 저장 |
| **AQ** | Aphasia Quotient (WAB 실어증 검사에서 유래). 세션 총점, 100점 만점 정수 |

---

## 2. 세션 구조

| 항목 | 확정 내용 |
|------|-----------|
| 세션 = 테마 1개 | TEST / HOSPITAL / CAFE. "오늘의 학습"은 **유저가 선택하지 않고 서버가 랜덤 선택** |
| 턴 구성 | 평가지표 4종 × 각 2회 = **8턴 (무작위 순서)** + STORYTELLING **4~8턴** |
| 8턴 생성 | 세션 입장 시 서버가 **한 번에 전부 선생성** (AI 컨테이너 일괄 생성 1회 호출). 병렬화는 Redis+Celery 도입 후 |
| 이야기 턴 | 선생성 불가 — 진행 중 **한 턴씩 생성** |
| 총 턴 수 | 12~16턴 (8 + 4~8) |

---

## 3. 세션 진행 흐름

```
[오늘의 학습 입장]
  → 테마 랜덤 선택 → 세션 생성 (LEARNING_SESSION INSERT)
  → AI 컨테이너 POST /sessions (8문제 일괄 생성)
  → TURN 8행 INSERT (로컬 turnId 리매핑) + AI TTS OCI 적재
  → [로딩 화면 대기]

[문제풀이 8턴] 순차 진행
  → 각 턴: 답안 제출 시마다 채점 요청 (제출자는 결과 대기 없이 다음 턴 진행)
  → LISTEN: 즉시 채점(백엔드 자체) / NAMING·SHADOWING·SELF_TALK: 컨테이너 동기 채점

[이야기 턴 4~8턴] (조기종료 버튼 있음)
  → 첫 턴: 8턴 결과 + 유저 정보를 맥락으로 AI가 먼저 대화 개시
  → 이후: context 배열 기반 대화 진행
  → 8턴 하드캡: 유저가 8턴째 대화를 마쳤을 때 AI의 마지막 응답(9번째 응답)으로 마무리

[세션 종료] (정상 마무리 / 조기종료 모두)
  → AI 컨테이너 POST /report → sessionAQ + 피드백 6종 수신
  → 결과 화면 로딩 대기 후 리포트 표시
  → 공유폴더 음성 파일 정리
```

### 채점 결과 수신 방식 (확정)

- **클라 ↔ BE**: 논블로킹 제출 — 유저는 채점 결과를 기다리지 않고 계속 진행
- **BE ↔ 컨테이너**: 동기 REST (`POST /answer/*`) — 응답받아 TURN.score / VOICE_RECORD 적재
- 이야기 턴이 채점 처리시간을 벌어주는 구조 → 이야기 턴 동안 대부분의 채점 완료
- 턴당 채점 결과는 **세션 결과 화면에서 일괄 표시** (턴당 피드백은 없음)
- ⚠️ pending: 이 부분 상세 수신 방식은 팀장 상의 후 재통보 예정

---

## 3. 타입별 상호작용 매트릭스

| 타입 | AI 음성 | 유저 입력 | 유저 음성 | 발화지표 | 채점 | 이미지 |
|------|---------|-----------|-----------|----------|------|--------|
| **LISTEN** (알아듣기) | O (TTS 질문) | **화면 선택지 탭** (발화 아님) | X | X | 백엔드 자체 (정답 100/오답 0) | 선택지 2~4개 (이미지/텍스트 혼합 가능) |
| **NAMING** (이름대기) | O | 발화 (STT) | O | O | 컨테이너 (절대정답) | 이미지 1개 |
| **SHADOWING** (따라말하기) | O | 발화 (따라말하기) | O | O | 컨테이너 (원문 대비) | X |
| **SELF_TALK** (스스로말하기) | O | 발화 (상황 묘사) | O | O | 컨테이너 (태그 비교) | 상황 이미지 1개 |
| **STORYTELLING** (이야기하기) | **X** (지연 우려, 향후 추가 가능) | 음성 (STT만) | O (기록만) | **X** | **X** | X |

### 타입별 상세

**LISTEN** — AI가 TTS로 문제를 읽어주면 화면의 선택지(2~4개, 이미지 또는 텍스트)를 **탭으로 선택**. 예: "사과를 고르세요", "오늘은 금요일인가요?". `correct_value`/`selected_value` = 선택지 ref. 선택지 이미지는 TURN_IMAGE에 order로. 유저 음성·VOICE_RECORD 행 없음.

**NAMING** — 이미지 1개 보고 발화로 답 (예: "마우스"). 절대정답 존재. 힌트는 **유저가 요청 버튼을 눌렀을 때** 공개 — 항상 **의미단서 → 조음단서 고정 순서**, 1개씩, 공개될 때마다 감점 (감점 수식은 컨테이너 내부 루브릭 — 백엔드 비관여). 힌트 2종(SEMANTIC_CUE/ARTICULATORY_CUE)은 **반드시 각 1개씩 존재**. `hints_shown`에 노출 수 기록 (0 허용).

**SHADOWING** — AI TTS 문장을 따라 말함. 이미지 미사용. 정답 Text를 백엔드가 보유 (`problemContext`로 컨테이너 전달).

**SELF_TALK** — 단일 사물이 아닌 "여러 일이 벌어지는 상황" 이미지. 유저가 상황을 묘사하면 **tags.json 내용과 유저 발화(STT)를 비교**해 채점. tags.json은 백엔드가 OCI에서 읽어 **내용 그대로** 컨테이너에 전달 (파싱 불필요 — 규약대로 업로드됨). `correct_value` = NULL.

**STORYTELLING** — 상황극 자유대화. 채점 없음. 첫 턴은 8턴 결과 + 유저 정보 기반으로 AI가 먼저 첫 대사. 유저 발화는 STT → `TURN.answer_text`, AI 발화 → `TURN.prompt_text`. AI 음성/발화지표 없음 (DB 구조는 확장 대비 유지).

---

## 4. 힌트 (NAMING)

| 항목 | 확정 내용 |
|------|-----------|
| 저장 | OCI hint.json 폐지 → **DB 컬럼 직접 저장** (`IMAGE_RESOURCE.SEMANTIC_CUE` / `ARTICULATORY_CUE`, nullable) |
| 공개 방식 | 유저가 힌트 요청 버튼 클릭 → 백엔드가 의미단서→조음단서 순으로 1개 반환 (동기) + `HINTS_SHOWN` 증가 |
| 존재 보장 | 2종 반드시 각 1개씩 (NAMING용 이미지는 cue 컬럼 필수 데이터) |
| 감점 | 공개마다 감점 — 수식은 컨테이너 루브릭 (계약서 `hintCount`로 전달) |

## 5. 리포트 (세션 결과)

| 항목 | 확정 내용 |
|------|-----------|
| 저장 위치 | **REPORT 테이블 신설 안 함** → `LEARNING_SESSION`에 통합: `AQ` + 피드백 6컬럼 (`LISTEN_FEEDBACK` / `NAMING_FEEDBACK` / `SHADOWING_FEEDBACK` / `SELF_TALK_FEEDBACK` / `TALK_FEEDBACK` / `TOTAL_FEEDBACK`) |
| 생성 시점 | 세션 종료 시점(정상·조기 모두)에 AI 컨테이너에 생성 요청 → 응답 수신 → 세션 컬럼 UPDATE |
| 표시 내용 | 턴별 채점 결과 점수( TURN 조회) + 지표별 피드백 4종 + AI 대화 피드백 + 종합 피드백 |
| AQ | 100점 만점 정수 (소수점 올림은 컨테이너 책임) |
| 결과 화면 | 리포트 생성 완료까지 로딩 대기 |
| 재시도 | 생성 실패 시 — 고도화 과제 (pending) |

## 6. 실력 산정 (대시보드)

| 항목 | 확정 내용 |
|------|-----------|
| 산정식 | **최근 20개 세션 중 지표별 상위 10개 세션의 평균** |
| userAQ | 동일식 (AQ 기준) — `/sessions` 요청 시 백엔드가 산정해 전달. 세션 0개 → `null` (추후 가입 설문으로 임시 AQ 주입) |

## 7. 상태 관리 / 이어하기

| 항목 | 확정 내용 |
|------|-----------|
| `LEARNING_SESSION.status` | IN_PROGRESS / COMPLETED / (폐기 등) |
| `TURN.status` | **PENDING**(출제·미풀이) / **SUBMITTED**(답안 제출) / **SCORED**(채점 완료) |
| 이어하기 | 세션 재입장 시 TURN.status로 진행 위치 판별 → 이어하기 제공 |
| 폐기·신규 | 미완료 세션 폐기 + 신규 생성 옵션도 제공 |
| 공유폴더 정리 | 세션 종료/폐기 시점에 삭제 |

## 8. 세션 생성 시퀀스 (8턴 INSERT 전략)

```
POST /sessions (BE → 컨테이너): 테마/이미지리스트/유저정보/userAQ 전달
  ← problemList 8개 (turnId는 컨테이너 로컬 번호 1~8)
백엔드: TURN 8행 INSERT (시퀀스로 실제 PK 발급 — UPDATE 불필요)
  · 로컬 turnId 순서 → turn_number
  · type → content_type / passage → prompt_text
  · listen correct/options → correct_value / choices_json
  · naming correct → correct_value
TTS: 공유폴더 {로컬턴ID}_ai.mp3 → 실제 TURN PK로 리네임 → OCI 적재
```

> 근거 (ADR-006): 턴 선 INSERT 시 어느 턴이 어떤 타입/문제를 가질지 알 수 없어 UPDATE가 2배 발생. 응답 수신 시점에 리매핑하면 INSERT 1회로 해결. 컨테이너는 자기가 만든 문제를 기억하지 않음.

## 9. pending (미확정)

| 항목 | 상태 |
|------|------|
| 턴당 채점 결과 수신 방식 상세 | 팀장 상의 후 재통보 |
| 리포트 생성 실패 재시도 | 고도화 과제 |
| BE↔컨테이너 타임아웃/에러 규약 | 고도화 과제 (데모 단계 규약 없음) |
| STORYTELLING AI 음성(TTS) 추가 | 향후 확장 — DB 구조는 유지 |
| userAQ 가입 설문 주입 | 향후 (첫 가입 시 임시 AQ 계산) |

## 10. 변경 이력

| 버전 | 날짜 | 내용 |
|------|------|------|
| v1.0 | 2026-09-02 | 초안 확정 — 기획 회의 전체 확정사항 반영 |