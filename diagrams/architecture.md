# 시스템 아키텍처 구조도 (v1.0 — 2026-09-09)

> **기준:** `02_Architecture.md` v2.1 §3 · `03a_AI_Container_API_Reference.md` v1.11 §0 · ADR-005 (음성 3단계 저장)
> **실측 근거 (2026-09-09 VM 실물):** Deployment compose — oracle-db volumes `/mnt/db_data:/opt/oracle/oradata` bind mount, `/mnt/db_data` = `/dev/sdb ext4 50GB` 블록 볼륨 · AI 컨테이너 `main.py`/`hf_stt.py`/`config.py` — STT=Whisper large-v3-turbo + LoRA 파인튜닝 가중치(lora_v2.pt), TTS=OpenAI API(고정 프리셋 화자), LLM=Ollama Cloud API

```mermaid
graph TB
    subgraph Client["Android 클라이언트"]
        A["덕담 앱<br/>Kotlin Native"]
    end

    subgraph Firebase["인증"]
        F["Firebase Auth<br/>Google OAuth2"]
    end

    subgraph OCI_VM["부트캠프 VM — Docker Compose (XEPDB1)"]
        direction TB
        N["nginx :80<br/>Reverse Proxy<br/>client_max_body_size 50m"]
        S["Spring Boot API :8080<br/>Kotlin 모놀리틱<br/>자체 JWT (Access 15분 / Refresh)"]
        D[("Oracle XE 21c :1521<br/>DBMS = 컨테이너")]
        BV[("블록 볼륨 /dev/sdb ext4<br/>/mnt/db_data<br/>DB 데이터 영구 저장")]
        AI["AI 컨테이너 :8000<br/>FastAPI<br/>STT · LLM · TTS · 채점 · 리포트"]
        SH[["공유폴더 (Docker volume)<br/>m4a/mp3 파일 교환<br/>— HTTP 전송 없음, 경로 문자열만"]]
        ADM["관리자 페이지<br/>FastAPI + Jinja2"]
    end

    subgraph OCI_Object["OCI Object Storage"]
        O1[("bucket-team545-userfiles<br/>음성 원본 · 프로필<br/>영구 저장 (ADR-005 ②단계)")]
        O2[("bucket-team545-problemfiles<br/>문제 이미지 · tags.json")]
    end

    subgraph Models["AI 모델"]
        M1["LLM — Ollama Cloud API"]
        M2["STT — 로컬 Whisper large-v3-turbo<br/>+ LoRA 파인튜닝 가중치(lora_v2)<br/>CPU 멀티스레드"]
        M3["TTS — OpenAI TTS API<br/>(고정 프리셋 화자 — 클로닝 미지원)"]
    end

    A -->|"HTTPS :80"| N
    N --> S
    A -.->|"Google ID Token 발급"| F
    F -.->|"ID Token 검증 → 자체 JWT 발급"| S
    S -->|"JPA"| D
    D -->|"데이터파일 bind mount<br/>/mnt/db_data:/opt/oracle/oradata"| BV
    ADM -->|"app=RO / admin=RW 계정 분리"| D

    S <-->|"동기 REST JSON<br/>/sessions · /answer/* · /aichat · /report/*"| AI
    S ---|"userVoicePath / ttsPath<br/>경로 교환"| SH
    SH <-->|"음성 파일 실물 교환<br/>(유저 m4a ↓ · TTS mp3 ↑)"| AI

    S -->|"영구 적재 — 원본 m4a 업로드"| O1
    O1 -->|"원본 읽기"| S
    S -->|"프록시 스트리밍<br/>GET /api/v1/voice/{id} (JWT)"| A
    O2 -->|"이미지 · tags.json 원문"| S

    AI --- M1
    AI --- M2
    AI --- M3
```

## 발표용 포인트

1. **음성 = HTTP 전송 없음** — BE↔AI컨테이너는 공유폴더 경로 문자열만 교환, 실물 파일은 Docker volume 경유 (03a §0)
2. **DBMS는 컨테이너, 데이터는 블록 볼륨** — Oracle XE 21c 컨테이너 + OCI 블록 볼륨(`/dev/sdb`) bind mount로 영구 저장
3. **음성 3단계 저장 (ADR-005)** — ①공유폴더(임시 교환) → ②OCI 버킷(영구 원본) → ③BE 프록시 스트리밍(JWT 인증)
4. **인증** — Firebase Google OAuth2 ID Token 검증 → 자체 JWT 발급(Access 15분/Refresh)
5. **AI 모델 스택(실측)** — LLM=Ollama Cloud API · STT=로컬 Whisper large-v3-turbo+LoRA(lora_v2) · TTS=OpenAI TTS API
