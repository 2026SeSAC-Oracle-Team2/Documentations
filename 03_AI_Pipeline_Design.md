# AI 파이프라인 설계서 (AI Pipeline Design Document)

## 1. 목적 및 범위

본 문서는 SRS 및 시스템 설계서에 기술된 AI 음성 대화 흐름의 **상세 파이프라인**을 정의한다.

- **STT**: OpenAI Whisper API (LLM 컨테이너 내부 호출)
- **발화 채점**: OCI 내 **채점 컨테이너** (내부 구현 TBD)
- **대화 진행**: OCI 내 **LLM 컨테이너** (Python FastAPI + LangGraph + GPT-4o-mini API)
- **TTS**: OpenAI TTS (tts-1, LLM 컨테이너 내부 호출)
- **실행 방식**: Spring Boot가 LLM 컨테이너 호출 → LLM 컨테이너 내부에서 STT→채점→LLM→TTS 순차 처리 → Spring Boot가 결과 취합 후 WebSocket으로 푸시

> **📌 리포지토리 구조:** AI 컨테이너는 각각 별도 GitHub 리포지토리로 관리된다.
> - `SeSAC_llm_container` — LLM 컨테이너 (STT + LLM + TTS 통합, Python/FastAPI)
> - `SeSAC_scoring_container` — 채점 컨테이너 (Python/FastAPI)
> - `SeSAC_deployment` — Docker Compose 오케스트레이션 (두 컨테이너를 포함한 전체 서비스 정의)

---

## 2. 전체 파이프라인 흐름

### 2.1 턴당 처리 흐름

```mermaid
graph LR
    subgraph Android
        A[마이크 녹음 종료]
    end

    subgraph OCI_API[OCI — Spring Boot API]
        B[음성 파일 수신]
        C[Object Storage 저장]
        D[LLM 컨테이너 호출]
        E[결과 취합]
        F[DB 저장]
    end

    subgraph OCI_AI[OCI — AI 컨테이너]
        LLM[llm-container
STT + LLM + TTS]
        SC[scoring-container
발화 채점]
    end

    subgraph OpenAI[OpenAI API]
        W[Whisper API]
        T[TTS tts-1]
        G[GPT-4o-mini]
    end

    subgraph DB[Oracle DB]
        I[Turn / Score / Problem 저장]
    end

    A -->|binary/MP3
최대 30초| B
    B -->|파일| C
    C -->|URL| D
    D -->|음성 파일 URL + 세션 컨텍스트| LLM
    LLM -->|Whisper API 호출| W
    W -->|stt_text| LLM
    LLM -->|stt_text + 문제 + 컨텐츠 타입| SC
    SC -->|점수 + 5개 지표 + 피드백| LLM
    LLM -->|대화 컨텍스트| G
    G -->|다음 문제/피드백 텍스트| LLM
    LLM -->|TTS 변환| T
    T -->|음성 binary| LLM
    LLM -->|STT 텍스트 + 피드백 + TTS URL + 점수| E
    E -->|WebSocket EVENT| A
    E -->|INSERT| I
```

### 2.2 세션 종합 보고서 생성 흐름

```mermaid
graph LR
    subgraph Android
        A[세션 종료 요청]
    end

    subgraph OCI_API[OCI — Spring Boot API]
        B[세션 종료 수신]
        C[DB에서 턴 데이터 조회]
        D[채점 컨테이너 호출]
        E[보고서 저장]
    end

    subgraph OCI_AI[OCI — AI 컨테이너]
        SC[scoring-container
세션 종합 채점]
    end

    subgraph DB[Oracle DB]
        I[Turn / Score / Report 저장]
    end

    A -->|세션 ID| B
    B -->|조회| C
    C -->|모든 턴 + 점수 데이터| D
    D -->|세션 데이터| SC
    SC -->|종합 보고서
5개 지표별 점수 + 종합 점수| D
    D -->|INSERT| I
    E -->|보고서 데이터| A
```

---

## 3. 각 단계 상세 규격

### 3.1 LLM 컨테이너 — 통합 AI 처리

| 항목 | 내용 |
|------|------|
| **엔드포인트** | `POST http://llm-container:8000/api/v1/process` (Docker 내부 네트워크) |
| **인증** | 내부 네트워크이므로 API Key 불필요 (Spring Boot IP 화이트리스트) |
| **입력** | 음성 파일 URL + 세션 컨텍스트 |
| **내부 처리** | 1) Whisper API 호출 → STT 2) STT 텍스트를 scoring-container에 전달 → 턴당 채점 3) LangGraph + GPT-4o-mini → 다음 문제/피드백 생성 4) TTS(tts-1) → 음성 변환 |
| **출력** | STT 텍스트, 피드백 텍스트, TTS 음성 URL, 5개 평가지표별 점수, 턴 번호 |
| **타임아웃** | 20초 (Whisper 5s + 채점 5s + LLM 5s + TTS 5s) |
| **재시도** | 1회 (connection error 시), exponential backoff 2s |

#### 입력 스키마

```json
{
  "voice_url": "https://objectstorage.oraclecloud.com/.../user-uuid/session-uuid/turn-3.mp3",
  "session_id": "session-uuid",
  "user_id": "user-uuid",
  "turn_number": 3,
  "content_type": "A",
  "original_question": "오늘의 문제 텍스트",
  "session_context": {
    "session_type": "DAILY",
    "turn_history": [
      {"speaker": "AI", "content": "안녕하세요! 오늘은 '주말에 뭐 했어요?'라는 주제로 대화해볼까요?"},
      {"speaker": "USER", "content": "주말에 친구랑 공원에 갔어요"}
    ]
  }
}
```

#### 출력 스키마

```json
{
  "stt_text": "주말에 친구랑 공원에 갔어요",
  "feedback_text": "문장 구성은 좋았어요! '친구랑'이라는 표현을 사용한 점이 자연스럽습니다.",
  "tts_url": "https://objectstorage.oraclecloud.com/.../user-uuid/session-uuid/tts-3.mp3",
  "next_question": "공원에서 무엇을 했나요? 한 문장으로 말해보세요.",
  "scores": {
    "metric_1": 75,
    "metric_2": 80,
    "metric_3": 65,
    "metric_4": 70,
    "metric_5": 72
  },
  "overall_score": 72.4,
  "turn_number": 3,
  "is_final": false
}
```

> **⚠️ `scores` 구조는 채점 기준 확정 후 조정됨.** 현재는 5개 평가지표(지표1~5)를 가정.

---

### 3.2 채점 컨테이너 — 턴당 채점

> **⚠️ 내부 구현 TBD.** 아래는 LLM 컨테이너 ↔ 채점 컨테이너 간 **HTTP 인터페이스 규격**만 정의.

| 항목 | 내용 |
|------|------|
| **엔드포인트** | `POST http://scoring-container:8001/api/v1/score` (Docker 내부 네트워크) |
| **호출자** | llm-container (턴당 채점 시) / Spring Boot (세션 종합 채점 시) |
| **입력 JSON** | 아래 스키마 참조 |
| **출력 JSON** | 아래 스키마 참조 |
| **타임아웃** | 10초 |
| **재시도** | 0회 (실패 시 점수 없이 피드백만 전달하는 폴백) |

#### 턴당 채점 — 입력 스키마

```json
{
  "stt_text": "사용자가 말한 내용",
  "original_question": "오늘의 문제 텍스트 또는 주제",
  "user_id": "user-uuid",
  "turn_number": 3,
  "session_id": "session-uuid",
  "content_type": "A",
  "session_type": "DAILY"
}
```

#### 턴당 채점 — 출력 스키마

```json
{
  "scores": {
    "metric_1": 75,
    "metric_2": 80,
    "metric_3": 65,
    "metric_4": 70,
    "metric_5": 72
  },
  "overall_score": 72.4,
  "feedback_text": "문장 구성은 좋았으나 '그리고'라는 접속사가 3번 반복되었어요. 다양한 표현을 시도해보세요.",
  "parameters": {
    "fluency": 75,
    "vocabulary": 65,
    "sentence_structure": 80
  }
}
```

#### 세션 종합 채점 — 입력 스키마

```json
{
  "session_id": "session-uuid",
  "user_id": "user-uuid",
  "session_type": "DAILY",
  "turns": [
    {
      "turn_number": 1,
      "content_type": "A",
      "stt_text": "...",
      "scores": {
        "metric_1": 75,
        "metric_2": 80,
        "metric_3": 65,
        "metric_4": 70,
        "metric_5": 72
      }
    },
    {
      "turn_number": 2,
      "content_type": "B",
      "stt_text": "...",
      "scores": {
        "metric_1": 70,
        "metric_2": 75,
        "metric_3": 80,
        "metric_4": 65,
        "metric_5": 78
      }
    }
  ]
}
```

#### 세션 종합 채점 — 출력 스키마

```json
{
  "session_id": "session-uuid",
  "summary_scores": {
    "metric_1": 72.5,
    "metric_2": 77.5,
    "metric_3": 72.5,
    "metric_4": 67.5,
    "metric_5": 75.0
  },
  "overall_score": 73.0,
  "feedback_summary": "전반적으로 문장 구성이 안정적입니다. 다양한 접속사 사용과 구체적 표현을 연습해보세요.",
  "content_type_breakdown": {
    "A": {"average": 72.0, "count": 2},
    "B": {"average": 74.0, "count": 1}
  }
}
```

> **⚠️ `summary_scores`의 5개 지표는 평가지표(지표1~5)를 가정.** 컨텐츠 타입별 가중치는 기획 확정 후 적용.

---

### 3.3 STT — OpenAI Whisper API (LLM 컨테이너 내부)

| 항목 | 내용 |
|------|------|
| **엔드포인트** | `POST https://api.openai.com/v1/audio/transcriptions` |
| **인증** | `Authorization: Bearer ***` |
| **입력** | `multipart/form-data` — `file` (MP3/WAV, ≤25MB), `model` = `whisper-1`, `language` = `ko` |
| **출력** | `{ "text": "사용자가 말한 내용..." }` |
| **타임아웃** | 10초 |
| **재시도** | 1회 (connection error 시), exponential backoff 2s |
| **비용** | $0.006/분 |
| **녹음 제한** | 최대 30초 (클라이언트에서 제한). 25MB 제한으로 사실상 문제 없음 |

---

### 3.4 대화 진행 — LangGraph + GPT-4o-mini (LLM 컨테이너 내부)

| 항목 | 내용 |
|------|------|
| **LLM 엔진** | GPT-4o-mini API (외부 REST 호출) |
| **프레임워크** | LangGraph (Python) |
| **입력** | 대화 컨텍스트(전체 Turn) + 현재 STT 텍스트 + 채점 결과 |
| **출력** | 다음 문제/대화 텍스트, 힌트, 종료 여부 |
| **타임아웃** | 10초 |
| **재시도** | 1회 (GPT-4o-mini 타임아웃 시) |
| **비용** | GPT-4o-mini: 약 $0.0003/1K input token, $0.0012/1K output token |

#### LangGraph 입력

```json
{
  "session_id": "session-uuid",
  "user_id": "user-uuid",
  "content_type": "A",
  "turn_history": [
    {"speaker": "AI", "content": "오늘은 '주말에 뭐 했어요?'라는 주제로 대화해볼까요?"},
    {"speaker": "USER", "content": "주말에 친구랑 공원에 갔어요"},
    {"speaker": "AI", "content": "좋아요! 공원에서 무엇을 했나요?"},
    {"speaker": "USER", "content": "그리고 그리고 산책했어요"}
  ],
  "previous_summary": "사용자는 주말 활동에 대해 간단히 답변할 수 있음. 접속사 반복이 관찰됨.",
  "current_stt": "그리고 그리고 산책했어요",
  "current_scores": {
    "metric_1": 65,
    "metric_2": 70,
    "metric_3": 60,
    "metric_4": 68,
    "metric_5": 72
  },
  "turn_number": 4
}
```

#### LangGraph 출력

```json
{
  "next_content": "산책하니까 기분이 어땠어요? 한 문장으로 말해보세요.",
  "content_type": "A",
  "hint": "기분, 날씨, 친구 중 하나를 포함해보세요",
  "is_final": false,
  "session_end_reason": null
}
```

#### 세션 종료 조건

| 조건 | 설명 |
|------|------|
| 각 컨텐츠 타입별 최소 1회 완료 | 3개 컨텐츠(A, B, C)를 모두 순회하면 종료 제안 |
| `is_final = true` | LLM이 대화 자연스럽게 마무리 판단 |
| 사용자 "오늘은 여기까지" | 명시적 종료 요청 |

---

### 3.5 TTS — OpenAI TTS (tts-1, LLM 컨테이너 내부)

| 항목 | 내용 |
|------|------|
| **엔드포인트** | `POST https://api.openai.com/v1/audio/speech` |
| **인증** | `Authorization: Bearer ***` |
| **입력** | `application/json` — `{ "model": "tts-1", "input": "피드백 텍스트", "voice": "alloy", "speed": 1.0 }` |
| **출력** | `audio/mpeg` 바이너리 (MP3, 24kbps) |
| **타임아웃** | 8초 |
| **재시도** | 1회 |
| **비용** | $0.015/1K자 |
| **한국어 음성** | `voice`: `alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer` 중 선택 |

> **⚠️ 한국어 품질 리스크 (P0 필수 확인):** `tts-1`의 voice는 영어 최적화 음성으로, 한국어 발화 품질이 기대 이하일 수 있다. **8/22까지 한국어 샘플 A/B 테스트로 확정**한다. 품질 미달 시 대안: **Azure Neural TTS**(ko-KR-SunHi 등) 또는 **네이버 Clova Voice**.

#### TTS 음성 저장 흐름

```mermaid
sequenceDiagram
    participant LLM as LLM 컨테이너
    participant OpenAI as OpenAI TTS
    participant O as Object Storage

    LLM->>OpenAI: POST /v1/audio/speech (피드백 텍스트)
    OpenAI-->>LLM: MP3 binary
    LLM->>O: PUT {user-uuid}/{session-id}/tts-{turn-number}.mp3
    O-->>LLM: 저장 URL
    LLM-->>Spring Boot: TTS URL 포함하여 반환
```

---

## 4. LangGraph 구조 상세

### 4.1 상태 그래프 (State Graph)

```mermaid
graph TD
    A[START] --> B{input_parse}
    B --> C[stt_process]
    C --> D[scoring_call]
    D --> E[context_load]
    E --> F[problem_generate]
    F --> G{session_check}
    G -->|미완료| H[feedback_generate]
    G -->|완료| I[session_end]
    H --> J[tts_generate]
    I --> K[summary_generate]
    K --> J
    J --> L[output_format]
    L --> M[END]
```

### 4.2 노드 정의

| 노드명 | 역할 | GPT-4o-mini 프롬프트 개요 |
|--------|------|---------------------------|
| `input_parse` | 음성 파일 URL 파싱 및 컨텍스트 정리 | "음성 파일 URL과 세션 정보를 파싱" |
| `stt_process` | Whisper API 호출 | "사용자 음성을 텍스트로 변환" |
| `scoring_call` | 채점 컨테이너 호출 | "STT 결과를 채점 컨테이너에 전달하여 5개 평가지표 점수 수신" |
| `context_load` | DB에서 이전 턴 + 요약 로드 | Conversation 테이블의 전체 Turn을 system message로 구성 |
| `problem_generate` | 다음 문제/대화 생성 | "사용자의 수준과 이전 대화를 고려하여 다음 발화 연습 문제를 생성. content_type에 맞춰서." |
| `feedback_generate` | 사용자 발화 피드백 | "사용자의 발화를 평가하고 격려 및 구체적인 개선 방향 제시" |
| `session_check` | 세션 종료 여부 판단 | "모든 컨텐츠 타입(A, B, C)을 순회했으면 종료. 아니면 계속." |
| `session_end` | 세션 종료 처리 | 종료 인사 + 보고서 생성 트리거 |
| `summary_generate` | 세션 요약 생성 | "이번 세션의 사용자 발화 특징을 2~3문장으로 요약" |
| `tts_generate` | TTS 변환 | "피드백 텍스트를 음성으로 변환" |
| `output_format` | 출력 JSON 포맷팅 | {"stt_text", "feedback_text", "tts_url", "next_question", "scores", "is_final"} |

### 4.3 엣지 조건

```python
# pseudo-code (LangGraph)
def session_check(state: ConversationState):
    # 컨텐츠 타입별 최소 1회 완료 여부 확인
    completed_types = set(turn.content_type for turn in state.turn_history)
    if len(completed_types) >= 3:  # A, B, C 모두 완료
        return "session_end"
    if state.user_explicit_end:
        return "session_end"
    if state.turn_number >= 10:  # 최대 턴 수 제한
        return "session_end"
    return "feedback_generate"

graph.add_conditional_edges("session_check", session_check, {
    "session_end": "session_end",
    "feedback_generate": "feedback_generate"
})
```

---

## 5. Spring Boot ↔ AI 컨테이너 흐름

### 5.1 턴당 처리 시퀀스

```mermaid
sequenceDiagram
    actor U as 사용자
    participant A as Android
    participant WS as WebSocket
Note over API,DB: Spring Boot
    participant AS as @Async Thread Pool
    participant O as OCI Object Storage
    participant LLM as llm-container
    participant SC as scoring-container
    participant DB as Oracle XE

    U->>A: 녹음 종료 (최대 30초)
    A->>WS: POST /voice/upload (REST, binary)
    WS->>O: 음성 파일 저장
    O-->>WS: voice_url
    WS-->>A: {voice_record_id, storage_url}
    A->>WS: VOICE_UPLOAD {voice_record_id, storage_url}

    WS->>AS: processLlmPipeline()
    AS->>LLM: POST /api/v1/process (voice_url, session_context)
    LLM->>LLM: STT (Whisper API)
    LLM->>SC: POST /api/v1/score (stt_text, 문제, 컨텐츠 타입)
    SC-->>LLM: 점수 + 5개 지표 + 피드백
    LLM->>LLM: 다음 문제/피드백 생성 (GPT-4o-mini + LangGraph)
    LLM->>LLM: TTS 변환 (tts-1)
    LLM->>O: TTS 음성 저장
    O-->>LLM: tts_url
    LLM-->>AS: PipelineResult
    AS->>DB: Turn, Score 저장
    AS-->>WS: CompletableFuture<PipelineResult>
    WS-->>A: EVENT: TURN_RESULT {stt_text, feedback_text, tts_url, scores, next_question}
    A->>U: 피드백 음성 재생 + 텍스트 표시 + 점수 표시
```

### 5.2 세션 종합 보고서 시퀀스

```mermaid
sequenceDiagram
    actor U as 사용자
    participant A as Android
    participant S as Spring Boot API
    participant DB as Oracle XE
    participant SC as scoring-container

    U->>A: 세션 종료
    A->>S: POST /report/generate (세션 ID)
    S->>DB: 해당 세션의 모든 Turn, Score 조회
    S->>SC: POST /api/v1/score/session (세션 데이터)
    SC-->>SC: 5개 평가지표별 점수 산출 + 종합 점수(평균)
    SC-->>S: 종합 보고서 JSON
    S->>DB: Report 저장
    S-->>A: 보고서 데이터
    A->>U: 종합 보고서 화면 표시
```

### 5.3 WebSocket 이벤트 최종 명세

| 방향 | 이벤트명 | 페이로드 | 설명 |
|------|---------|---------|------|
| C → S | `SESSION_START` | `{content_type_id, session_type}` | 세션 시작 |
| C → S | `VOICE_UPLOAD` | `{voice_record_id, storage_url}` | 음성 업로드 완료 통지 (REST 업로드 후) |
| S → C | `TURN_RESULT` | `{stt_text, feedback_text, tts_url, scores, next_question, is_final}` | 턴 처리 완료 (STT + 채점 + LLM + TTS 통합) |
| S → C | `ERROR` | `{code, message}` | 처리 실패 |
| C → S | `USER_TEXT` | `{text}` | 사용자가 텍스트로 입력 (폴백) |
| C → S | `SESSION_END` | `{}` | 사용자 종료 요청 |
| S → C | `REPORT_READY` | `{report_id, overall_score, summary_scores}` | 보고서 생성 완료 |

---

## 6. 에러 처리 및 폴백 전략

| 에러 시나리오 | 폴백 동작 | 사용자 노출 |
|--------------|-----------|------------|
| **Whisper API 실패** (타임아웃/500) | 재시도 1회 → 실패 시 "음성을 다시 녹음해주세요" 안내 | "음성 인식에 실패했어요. 다시 말씀해주세요." |
| **채점 컨테이너 타임아웃** | 점수 null, feedback = "채점 중이에요. 계속 연습해보세요." | 점수 영역 숨김, 피드백만 표시 |
| **LLM 컨테이너 타임아웃** | 재시도 1회 → 실패 시 "잠시만 기다려주세요" + 고정 안내 문구 | "AI가 생각 중이에요..." → 5초 후 고정 메시지 |
| **TTS 실패** | 텍스트만 표시, 음성 재생 버튼 숨김 | 텍스트 피드백은 정상 표시 |
| **WebSocket 끊김** | 자동 재연결 (Exponential Backoff: 1s, 2s, 4s) | 재연결 중 "연결 중..." 표시 |
| **세션 중복 시작** | 기존 세션 자동 종료 + 새 세션 시작 | "이전 연습을 종료하고 새로 시작합니다." |
| **세션 종합 보고서 생성 실패** | 턴 데이터는 DB에 저장된 상태 유지, 사용자에게 "보고서 생성에 실패했어요" 안내 | "보고서를 생성할 수 없었어요. 학습 기록은 안전하게 저장되었습니다." |

---

## 7. 보안 및 비용

### 7.1 API 키 관리

| 키 | 위치 | 주입 방식 |
|----|------|----------|
| `OPENAI_API_KEY` | llm-container 환경변수 | `.env` 파일 → Docker Compose `environment` |
| `OPENAI_ORG_ID` | llm-container 환경변수 | 동일 |
| `FIREBASE_PROJECT_ID` | Spring Boot 환경변수 | `.env` 파일 → Docker Compose `environment` |

> `.env` 파일은 `.gitignore`에 포함. OCI VM에 직접 배포 시 `scp` 또는 GitHub Secrets로 전달.

### 7.2 예상 비용 산정 (MVP 기준)

| 항목 | 단위 비용 | 월간 예상 사용량 | 월간 비용 |
|------|----------|----------------|----------|
| Whisper API | $0.006/분 | 300분 (100회 × 3분) | **$1.8** |
| GPT-4o-mini | $0.0003/1K tok (in) | 500K input + 200K output | **$0.39** |
| TTS (tts-1) | $0.015/1K자 | 50K자 (100회 × 500자) | **$0.75** |
| **OpenAI 합계** | | | **약 $3/월** |
| OCI VM (4 OCPU/16GB) | 약 20만원/월 | 개발 중에는 껐다 켰다 | **실제 과금 ↓** |
| OCI Object Storage | 약 2만원/월 | 50GB 내외 | **2만원** |
| **총합** | | | **OpenAI 무시할 수준. OCI가 대부분.** |

> **결론**: OpenAI API 비용은 거의 무시 가능. OCI VM 과금이 대부분.

---

## 8. 변경 이력

| 버전 | 날짜 | 작성자 | 변경 내용 |
|------|------|--------|-----------|
| v0.1 | 2026-08-18 | 김윤혁 | 초안 작성 |
| v0.2 | 2026-08-18 | 김윤혁 | scoring-container 포트 8001로 통일. 음성 업로드 REST-first로 통일. TTS 한국어 품질 리스크 P0 승격. |
| v0.3 | 2026-08-19 | 김윤혁 | 컨테이너 구조 변경: conversation-llm+Whisper+TTS → **llm-container(통합)**. Spring Boot의 AI 진입점 유일화. 세션 종합 보고서 흐름 추가. 5개 평가지표 반영. 컨텐츠 3개(A/B/C) 반영. 녹음 30초 제한 추가. WebSocket 이벤트 통합(TURN_RESULT). |
| v0.4 | 2026-08-26 | - | 리포지토리 분리 반영: `SeSAC_llm_container`, `SeSAC_scoring_container` 별도 리포지토리 명시. `SeSAC_deployment` 오케스트레이션 리포지토리 참조 추가. |
