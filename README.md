# 이미지 분석 AI Agent (설계 단계)

카메라 화질 평가용 차트 이미지(raw/bmp/jpg)를 브라우저에서 올리면 차트 유형을 판별하고, 과제별 C DLL 계산(SFR 해상도 등)과 전체 이미지 기본 평가를 수행해 판정 보고서를 생성하는 **웹 앱 형태의 AI Agent**. 사내 LLM(GLM-5.3-Flash)은 백엔드가 OpenAI 호환 API로 호출하며, API가 없어도 수치·판정·보고서가 나오고 해석만 수동으로 붙여넣는 대체 경로를 둔다. 12시간 사내 해커톤을 목표로 한다.

현재 상태: 요구사항·설계 3차 초안, 승인 대기. 제품 코드는 아직 없다.

## 문서

| 문서 | 내용 |
|---|---|
| [docs/requirements.md](docs/requirements.md) | 요구사항, 인수 조건, 가정·질문·위험 (P0/P1/P2) |
| [docs/design.md](docs/design.md) | 시스템 구조, REST API·데이터 계약, LLM 연동 가이드, 12시간 타임라인 |
| [docs/hackathon-prompt-pack-web.md](docs/hackathon-prompt-pack-web.md) | 사내 LLM에 붙여넣을 빌드·런타임·운영 프롬프트 전문 (현행) |
| [docs/hackathon-prompt-pack.md](docs/hackathon-prompt-pack.md) | 이전 MCP 도구 서버 버전 프롬프트 팩 (대체됨, 이력 보존) |
| [docs/decisions.md](docs/decisions.md) | ADR 등록부 |
| [docs/work/20261011-image-analysis-agent/](docs/work/20261011-image-analysis-agent/) | ADR-001~004 |

## 구조 요약

```text
브라우저 (정적 HTML/JS)
   │ HTTP/JSON
FastAPI 백엔드 ── 작업 상태 기계(needs_input 확인 요청) ── 코어(로더 · 기본 평가 · DLL ctypes · 판정 · 보고서)
   └─ LlmClient (OpenAI 호환, 선택) : 판별 보조 · 서술 · 채팅        CLI (같은 코어)
```

LLM은 판별 보조와 해석만 맡고, 수치는 결정적 코드와 DLL이 계산한다.
