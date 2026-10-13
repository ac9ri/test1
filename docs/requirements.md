# REQ-image-analysis-agent: 이미지 분석 AI Agent 요구사항

> 문서 유형: `requirements`
> 작업 ID: `20261011-image-analysis-agent`
> 상태: `awaiting-approval`
> 기준선: `draft`
> 작성일: 2026-10-11
> 최종 갱신: 2026-10-13
> 관련 문서: [DESIGN-image-analysis-agent: 이미지 분석 AI Agent 설계](./design.md), [결정 등록부](./decisions.md)

## 요약

- 목적: 12시간 사내 해커톤에서 사내 LLM(GLM-5.3-Flash)을 쓰는 AI Agent를 **웹 애플리케이션**으로 만든다. 브라우저에서 raw·bmp·jpg 차트 이미지를 올리면 차트 유형 판별, SFR 해상도 등 과제별 C DLL 계산, 전체 이미지 기본 평가, 판정, 보고서 생성까지 끝난다.
- 현재 결론 또는 상태: 3차 초안. 2026-10-13 사용자 결정으로 목표가 MCP 도구 서버 개발에서 웹 개발로 바뀌었다. 웹 백엔드가 LLM을 API로 호출하는 구조를 기본으로 하되, API가 없어도 브라우저에서 수치 결과와 보고서가 나오고 해석 문장만 수동으로 붙여넣는 대체 경로를 P0에 포함한다. 승인 대기.
- 다음 행동: 사용자가 Q-07(사내 LLM API 유무·형식)에 답하고 승인하면 기준선 `v1`을 발행하고 [wf-implement](file:///C:/Users/JS/.claude/skills/wf-implement/SKILL.md)로 인계한다.

## 문서 연결

| 방향 | 관계 | 대상 문서 | 대상 항목 | 비고 |
|---|---|---|---|---|
| input | baseline | N/A — 루트 문서이며 선행 문서 없음 | — | 2026-10-11 요청, 10-11 1차 답변, 10-13 목표 변경이 입력 |
| output | refinement | [DESIGN-image-analysis-agent: 이미지 분석 AI Agent 설계](./design.md) | document | 이 요구사항을 구체화한 설계 |
| output | decision | [ADR-001: LLM과 결정적 코드의 역할 분리](./work/20261011-image-analysis-agent/ADR-001-llm-role-split.md) | FR-02, FR-03, FR-06, NFR-03 | 판별·해석은 LLM, 계산은 코드 |
| output | decision | [ADR-002: 과제 DLL 연동 방식](./work/20261011-image-analysis-agent/ADR-002-dll-worker-process.md) | FR-05, NFR-04 | ctypes 인프로세스 우선 |
| output | decision | [ADR-003: LLM 연동 방식 — 웹 백엔드의 OpenAI 호환 클라이언트와 수동 대체 경로](./work/20261011-image-analysis-agent/ADR-003-llm-integration.md) | FR-07, FR-13, NFR-01, NFR-02 | 실행 주체·연동 결정 |
| output | decision | [ADR-004: 웹 기술 스택 선택](./work/20261011-image-analysis-agent/ADR-004-web-stack.md) | FR-07, NFR-08, C-06 | FastAPI + 바닐라 JS |
| output | related | [결정 등록부](./decisions.md) | document | ADR 전체 목록 |

## 1. 문제와 목적

- 사실: 저장소에는 설계 문서만 있고 제품 코드·DLL·샘플 이미지는 없다.
- 사실(사용자 답변, 2026-10-11): 대표 분석은 SFR 해상도 평가이고 전체 이미지 평가도 원한다. 과제 DLL은 C 함수다. raw는 헤더 없는 베이어 덤프이며 과제별 고정이지만 입력받거나 추정하길 원한다. 사내 GLM은 "클로드 데스크톱 같은 사내 프로그램"을 통해 사용한다. 실행 주체·처리량은 미정이다. 샘플을 확보할 예정이다. 해커톤 12시간 제한이 있고 해커톤에서는 클로드를 쓸 수 없다.
- 사실(사용자 결정, 2026-10-13): 개발 목표가 MCP 도구 서버에서 **웹 개발**로 바뀌었다.
- 해석: Agent는 브라우저로 쓰는 웹 앱이 된다. 백엔드가 분석 파이프라인을 실행하고, LLM이 필요한 지점(판별 보조·해석·질의응답)은 백엔드가 사내 LLM API를 호출한다. API 존재와 형식은 미확인이므로 API 없이도 시연이 되는 대체 경로가 필요하다.
- 가정 A-01: "차트"는 카메라 화질 평가용 테스트 차트다(확정).
- 가정 A-10: 사내 LLM에 백엔드가 호출할 수 있는 HTTP API가 있고 OpenAI 호환 Chat Completions 형식이다. 틀리면 LLM 자동 호출 기능(FR-02 P1, FR-06 자동 서술, FR-12)은 비활성화되고 FR-13 수동 경로로 시연한다(Q-07).

**목적:** 해커톤 12시간 안에, 브라우저에서 이미지를 올리면 차트 판별 → 수치 계산 → 판정 → 보고서까지 자동으로 끝나고, 불확실한 지점(차트 유형·raw 사양)만 사용자에게 화면으로 확인받는 웹 Agent를 시연한다. 차트·과제·DLL은 설정으로 추가할 수 있어야 한다.

## 2. 대상 사용자와 호출 주체

| 구분 | 주체 | 호출 방식 | 우선순위 |
|---|---|---|---|
| 1차 | 해커톤 참가자(사용자)와 심사위원 | 로컬 PC의 브라우저에서 웹 UI 사용 | P0 |
| 2차 | 화질 평가 엔지니어 | REST API 또는 CLI | P0(CLI), P1(API 직접 호출) |
| 3차 | 검사 장비·배치 | REST API | P2, 범위 밖 |

## 3. 사용 시나리오

| ID | 시나리오 | 흐름 | 우선순위 |
|---|---|---|---|
| S-01 | 단일 이미지 분석(웹) | 업로드 → 진행 상태 표시 → 결과 화면(프리뷰·수치·판정) → 보고서 보기·다운로드 | P0 |
| S-02 | 폴더·다중 파일 분석 | 여러 파일 업로드 → 작업 목록 → 파일별 결과 | P1 |
| S-03 | raw 사양 확인 | 메타 없는 raw 업로드 → 추정 후보를 폼에 채워 제시 → 사용자 확인 → 재실행 | P1(추정), P0(수동 입력 폼) |
| S-04 | 판별 불확실 | 규칙 판별 실패 → LLM 후보(API 있을 때) 또는 카탈로그 목록 + 프리뷰 제시 → 사용자 선택 → 재실행 | P0(선택 UI), P1(LLM 후보) |
| S-05 | CLI 단독 실행 | `chart-agent analyze <경로>` → `result.json` + 템플릿 보고서 | P0 |
| S-06 | 신규 과제 추가 | 레지스트리 YAML + 어댑터 1파일 | P1 |
| S-07 | 결과 질의응답 | 결과 화면의 채팅에 "중앙 MTF가 왜 낮아?" → LLM이 도구로 재분석해 답변 | P1 |
| S-08 | LLM API 없음 | 결과 화면에서 "해석 프롬프트 복사" → 사내 채팅 프로그램에 붙여넣기 → 답을 화면에 붙여넣기 → 보고서 완성 | P0 |

## 4. 기능 요구사항

우선순위: P0 = 시연 필수, P1 = 시간이 남으면, P2 = 해커톤 이후.

| ID | 요구사항 | 우선순위 |
|---|---|---|
| FR-01 | 시스템은 `.bmp`, `.jpg/.jpeg`, `.raw`를 로드해야 한다. raw는 폭·높이·비트 깊이·베이어 패턴을 요청 인자 > 사이드카 > 레지스트리 과제 기본값 순으로 읽고, 파일 크기가 맞지 않으면 실패해야 한다. | P0 |
| FR-02 | 시스템은 차트 유형을 레지스트리 목록 중 하나 또는 `unknown`으로 판별하고 방법·신뢰도를 반환해야 한다. 사용자 지정 > 파일명 규칙 순으로 결정적 판별을 하고, 실패하면 LLM API가 있을 때 프리뷰로 LLM 판별을 시도하며, 신뢰도가 임계값 미만이면 후보와 프리뷰를 사용자에게 제시해 선택받아야 한다. | P0(규칙·선택 UI), P1(LLM 판별) |
| FR-03 | 수행할 분석 항목은 레지스트리가 결정한다. API와 LLM 도구는 레지스트리에 없는 분석 ID를 거부해야 한다. | P0 |
| FR-04 | 차트 유형과 무관하게 전체 이미지 기본 평가를 항상 수행해야 한다. 최소 항목: 해상도·채널·비트 깊이, 채널별 통계, 클리핑 비율, 선명도, 노이즈, 3×3 균일도. raw는 베이어 평면별 통계. | P0 |
| FR-05 | 과제 차트이면 레지스트리의 과제 C DLL 함수를 `ctypes`로 호출해야 한다. 첫 과제는 SFR이다. 실패 시 기본 평가 결과만으로 보고하고 실패를 명시한다. | P0 |
| FR-06 | 수치와 임계값으로 판정 표를 만들고, 서술 3절(요약·해석·권고)과 코드가 렌더링한 표를 합쳐 `report.md`를 생성해야 한다. 서술은 LLM API가 있으면 자동 생성하고, 없으면 템플릿 문장을 쓰며 FR-13 수동 경로를 제공한다. | P0 |
| FR-07 | 시스템은 (a) 브라우저 웹 UI, (b) 같은 기능의 REST API, (c) 같은 코어를 쓰는 CLI를 제공해야 한다. 웹 UI는 업로드, 진행 상태, 확인 요청, 결과·보고서 화면을 포함한다. | P0 |
| FR-08 | 차트 유형·과제·DLL·분석 항목·임계값·raw 기본값은 레지스트리 YAML로 추가·변경할 수 있어야 한다. | P0 |
| FR-09 | 판별이 `unknown`이거나 raw 사양이 불확실하면 작업은 `needs_input` 상태로 멈추고, 웹 UI가 후보를 제시해 사용자 입력을 받아 재개해야 한다. | P0 |
| FR-10 | raw 메타가 없으면 파일 크기·픽셀 통계로 폭·높이·비트·베이어를 추정하고 신뢰도·후보를 반환해야 한다. | P1 |
| FR-11 | 여러 파일을 한 번에 올려 작업 목록으로 처리하고 파일별 상태·결과를 보여줘야 한다. | P1 |
| FR-12 | 결과 화면의 채팅에서 사용자 질문에 LLM이 답해야 한다. LLM은 `get_result`, `run_analysis`, `get_catalog` 도구를 순차 호출할 수 있고, 수치는 도구 결과만 인용해야 한다. | P1 |
| FR-13 | LLM API가 없거나 실패하면 웹 UI는 해석 요청 프롬프트(수치·판정 표 포함)를 복사할 수 있게 하고, 사용자가 붙여넣은 서술로 보고서를 다시 생성할 수 있어야 한다. | P0 |

## 5. 비기능 요구사항

| ID | 요구사항 | 우선순위 |
|---|---|---|
| NFR-01 | 코어(로더·분석·DLL·판정·보고서)는 LLM·웹 프레임워크에 의존하지 않아야 한다. 웹 백엔드, CLI, LLM 클라이언트는 코어를 감싸는 층이다. | P0 |
| NFR-02 | 웹 서버는 기본으로 `127.0.0.1`에만 바인딩한다. 업로드 파일과 결과는 로컬 `out/`에만 저장한다. LLM에는 긴 변 1024px 프리뷰와 수치 표만 보낸다. API 키는 환경 변수로만 받고 로그·응답에 넣지 않는다. 인증은 해커톤 범위에서 두지 않는다. | P0 |
| NFR-03 | 동일 입력·설정이면 `result.json`의 수치 필드가 동일해야 한다. | P0 |
| NFR-04 | DLL 예외·반환 코드는 작업 실패로 기록되고 서버는 계속 동작해야 한다. 네이티브 크래시 격리는 P2다. | P0(예외), P2(격리) |
| NFR-05 | 성능 목표는 없다. 시연 기준으로 업로드부터 결과 화면까지 LLM 포함 2분 안에 끝나는지 리허설로 확인한다. 업로드 상한은 200MB. | P1 |
| NFR-06 | 작업마다 단계·소요 시간·판별·DLL 상태·LLM 호출(모델·토큰·지연)을 로그로 남긴다. | P1 |
| NFR-07 | 실제 DLL과 LLM API 없이도 개발·테스트할 수 있도록 mock DLL과 mock LLM 클라이언트를 둔다. | P0 |
| NFR-08 | 프런트엔드는 빌드 도구·외부 CDN 없이 정적 파일만으로 최신 크롬에서 동작해야 한다(오프라인 해커톤 대비). | P0 |

## 6. 범위와 범위 밖

**범위(P0·P1):** 코어(로더·raw 추정·규칙 판별·기본 평가·SFR DLL 어댑터·판정·보고서), 레지스트리, 웹 백엔드(REST API, 작업 관리), 웹 프런트엔드, LLM 클라이언트(판별·서술·채팅), 수동 대체 경로, CLI, 12시간 계획과 프롬프트.

**범위 밖:** 장비 제어·캡처, DLL 개발, 사용자 인증·권한, DB(파일 저장만), 외부 배포·HTTPS, 모델 파인튜닝, 사외 LLM, SFR 외 과제 어댑터, 데이터 그래프 해석, 병렬 처리, MCP 서버(설계 이력으로만 보존).

## 7. 제약과 의존성

| ID | 제약·의존성 |
|---|---|
| C-01 | 해커톤 약 12시간. P0(웹에서 업로드→보고서)를 6시간 안에 끝내는 것을 목표로 하고 못 끝내면 P1을 버린다. |
| C-02 | LLM 자동 호출은 사내 LLM API가 있을 때만 가능하다(A-10, Q-07). API가 없으면 FR-13 수동 경로로 시연한다. |
| C-03 | Windows 11, Python 3.11.9(사실). `numpy pillow pydantic pytest` 설치됨. `fastapi uvicorn python-multipart pyyaml opencv-python openai` 미설치. 해커톤 장소의 pip 가능 여부 확인 필요(Q-10). |
| C-04 | 과제 DLL은 C 함수(사실). 시그니처·비트 수·입력 형식 미확인(Q-08). 64비트 가정(A-07). |
| C-05 | GLM-5.3-Flash 공개 사양(2026-10-11 [Z.ai 문서](https://docs.z.ai/guides/vlm/glm-5.3-flash)): 이미지 입력·함수 호출·JSON 출력 지원, thinking 항상 켜짐. 사내 API가 이를 모두 노출하는지는 미확인. |
| C-06 | 웹 스택은 FastAPI + uvicorn 백엔드, 정적 HTML/CSS/JS 프런트엔드로 제한한다([ADR-004](./work/20261011-image-analysis-agent/ADR-004-web-stack.md)). |
| D-01 | 샘플 차트·이미지(사용자 확보 예정). SFR bmp/jpg 2장, raw 1장, 차트 아닌 이미지 1장이 P0 검증에 필요하다. |

## 8. 예외와 실패 시 기대 동작

| 상황 | 기대 동작 |
|---|---|
| 지원하지 않는 확장자·상한 초과 업로드 | HTTP 400과 메시지, 화면에 표시 |
| raw 메타 없음 | 추정(P1) 후 신뢰도 낮으면 `needs_input`(raw_meta) + 후보 폼. 추정 없으면 빈 폼 |
| raw 크기 불일치 | 작업 `failed`, 기대 크기 메시지 |
| 판별 `unknown` | LLM 판별(있을 때) → 여전히 불확실하면 `needs_input`(chart_type) + 프리뷰·후보 |
| DLL 실패 | `dll_status: failed`, 기본 평가 유지, 상태 `partial`, 화면에 원인 |
| LLM API 미설정·연결 실패·응답 파싱 실패 | 재시도 후 템플릿 서술로 보고서 생성, `narrative_source: template`, 수동 경로 버튼 표시 |
| LLM 채팅 도구가 없는 분석 ID 요청 | 도구가 오류와 허용 목록 반환, LLM이 수정 |
| 서버 재시작 | `out/` 폴더에서 작업 목록 복구(P1) |

## 9. 인수 조건

| ID | 입력·상황 | 행동 | 기대 결과 | 관련 | 우선순위 |
|---|---|---|---|---|---|
| AC-01 | bmp, jpg 샘플 | 로드 | 폭·높이·채널 일치 | FR-01 | P0 |
| AC-02 | raw + 메타 / 크기 불일치 raw | 로드 | 프리뷰·베이어 배열 / `E_RAW_SIZE_MISMATCH` | FR-01 | P0 |
| AC-03 | 확보한 샘플 전부 | 판별 | SFR은 규칙·LLM·선택으로 `sfr_iso12233`, 비차트는 `unknown` 또는 사용자 확인. 오판별 0건 | FR-02 | P0 |
| AC-04 | SFR 차트 + DLL(또는 mock) | 분석 | `project.results`에 반환값, `dll_status: ok` | FR-05 | P0 |
| AC-05 | 예외 mock DLL | 분석 | 기본 평가 포함, `dll_status: failed`, 서버 생존 | FR-05, NFR-04 | P0 |
| AC-06 | CLI 단독 실행 | 분석 | `result.json`과 템플릿 `report.md` | FR-06, FR-07 | P0 |
| AC-07 | 임의 결과 + 서술 | 보고서 | 수치 표가 `result.json`과 일치 | FR-06, NFR-03 | P0 |
| AC-08 | 레지스트리에만 유형·과제·mock 추가 | 분석 | 코어 변경 없이 동작 | FR-08 | P1 |
| AC-09 | 없는 분석 ID로 API·도구 호출 | 호출 | 오류와 허용 목록, 미실행 | FR-03 | P0 |
| AC-10 | 동일 입력 2회 | 수치 비교 | 완전 일치 | NFR-03 | P0 |
| AC-11 | 브라우저에서 SFR 샘플 업로드 1회 | 자동 진행 | 결과 화면에 프리뷰·판정 표가 나오고 `report.md`를 내려받을 수 있다 | FR-07, FR-06 | P0 |
| AC-12 | 사이드카 없는 raw | 추정 | 폭·높이·비트 일치, 베이어는 G 위치 일치 | FR-10 | P1 |
| AC-13 | LLM 미설정 상태에서 분석 | 수동 경로 | 해석 프롬프트 복사 → 서술 붙여넣기 → 보고서에 반영, `narrative_source: manual` | FR-13 | P0 |
| AC-14 | `unknown` 차트 업로드 | 확인 요청 | 작업이 `needs_input`에서 멈추고 후보 선택 후 재개되어 완료된다 | FR-09 | P0 |
| AC-15 | LLM API 설정 + 결과 화면 채팅 | 질의 | 답변이 도구 결과 수치만 인용하고 도구 호출 내역이 표시된다 | FR-12 | P1 |

## 10. 가정, 질문, 위험

### 가정

| ID | 가정 | 틀리면 영향 |
|---|---|---|
| A-01 | 차트는 화질 평가용 테스트 차트(확정) | — |
| A-07 | 과제 DLL은 64비트, 입력은 버퍼 또는 경로 | 32비트면 워커 격리(P2) 앞당김 |
| A-09 | 해커톤에서 pip 설치와 문서 반입 허용 | 휠 사전 준비, 의존성 축소 |
| A-10 | 사내 LLM API가 있고 OpenAI 호환이다 | LLM 자동 기능 비활성화, 수동 경로로 시연 |
| A-11 | 시연은 발표자 PC의 브라우저에서 한다 | 외부 접근이 필요하면 바인딩·방화벽 설정 추가 |

### 미해결 질문

| ID | 질문 | 왜 중요한가 |
|---|---|---|
| Q-07 | 사내 LLM API가 있는가? 있다면 엔드포인트 형식(OpenAI 호환?), 인증, 이미지 입력·`tools`·`response_format` 지원, 속도 제한 | LLM 자동 기능 P0.5 여부, 클라이언트 대체 전략 |
| Q-08 | SFR DLL 시그니처, 입력(경로/버퍼), 32/64비트, 헤더 유무 | 어댑터와 mock |
| Q-09 | 과제 raw 기본 사양 | 레지스트리 기본값, 추정 검증 |
| Q-10 | 해커톤 규칙: 사전 코드·문서 반입, pip·인터넷, 팀 인원 | 준비 범위 |
| Q-11 | 심사 방식: 발표자 PC 시연인지 심사위원이 직접 접속하는지 | 바인딩·동시 사용자 |

### 위험

| ID | 위험 | 대응 |
|---|---|---|
| RISK-01 | 유사 차트 오판별 | 규칙 우선, 신뢰도 임계값, 사용자 선택 UI |
| RISK-02 | DLL 크래시로 웹 서버 종료 | P0 예외 처리 + uvicorn 재시작 안내, P1에서 파이프라인을 CLI 서브프로세스로 실행하는 옵션, P2 워커 |
| RISK-03 | 사내 LLM API 없음 또는 기능 미지원 | FR-13 수동 경로를 P0로 두고, 클라이언트는 구조화 출력 3단계 대체 |
| RISK-04 | thinking 지연으로 결과 화면이 느림 | 판별·서술 2회 호출로 제한, `reasoning_effort` 낮춤, 진행 상태 표시 |
| RISK-05 | LLM 수치 왜곡 | 표는 코드 삽입, 프롬프트 인용 규칙, 서술 내 숫자 검증(P1) |
| RISK-06 | raw 추정 오류 | 크기 검증, 후보 폼, 프리뷰 확인 |
| RISK-07 | 12시간 안에 P0 미완 | 6시간 체크포인트, 프런트엔드는 기능 우선·꾸밈 후순위 |
| RISK-08 | 실제 DLL 지연 | mock 우선, 어댑터 1파일 교체 |
| RISK-09 | 프런트엔드 작성 시간 초과 | 단일 HTML + 바닐라 JS, 화면 4개(업로드·진행·확인·결과)로 제한 |

## 추적성

| 요구사항 | 설계 | 인수 조건 | 검증 |
|---|---|---|---|
| FR-01 | [DES-03](./design.md#des-03-imageloader와-raw-추정) | AC-01, AC-02 | TBD — wf-implement 계획 수립 시 VER 발행 |
| FR-02 | [DES-04](./design.md#des-04-chartclassifier), [DES-08](./design.md#des-08-llmclient) | AC-03, AC-14 | TBD — 동상 |
| FR-03 | [DES-05](./design.md#des-05-registry), [DES-10](./design.md#des-10-webbackend) | AC-09 | TBD — 동상 |
| FR-04 | [DES-06](./design.md#des-06-basicanalyzer) | AC-01, AC-10 | TBD — 동상 |
| FR-05 | [DES-07](./design.md#des-07-dlladapter) | AC-04, AC-05 | TBD — 동상 |
| FR-06 | [DES-09](./design.md#des-09-judge와-reportbuilder), [DES-08](./design.md#des-08-llmclient) | AC-06, AC-07 | TBD — 동상 |
| FR-07 | [DES-01](./design.md#des-01-cli), [DES-10](./design.md#des-10-webbackend), [DES-11](./design.md#des-11-webfrontend) | AC-06, AC-11 | TBD — 동상 |
| FR-08 | [DES-05](./design.md#des-05-registry) | AC-08 | TBD — 동상 |
| FR-09 | [DES-02](./design.md#des-02-pipeline), [DES-10](./design.md#des-10-webbackend), [DES-11](./design.md#des-11-webfrontend) | AC-14 | TBD — 동상 |
| FR-10 | [DES-03](./design.md#des-03-imageloader와-raw-추정) | AC-12 | TBD — 동상 |
| FR-11 | [DES-10](./design.md#des-10-webbackend) | — | TBD — 리허설 |
| FR-12 | [DES-12](./design.md#des-12-chatagent) | AC-15 | TBD — 동상 |
| FR-13 | [DES-10](./design.md#des-10-webbackend), [DES-11](./design.md#des-11-webfrontend) | AC-13 | TBD — 동상 |
| NFR-01 | [2. 시스템 구조](./design.md#2-시스템-구조) | AC-06, AC-11 | TBD — 동상 |
| NFR-02 | [6. 품질 속성](./design.md#6-품질-속성) | — | TBD — 코드 리뷰 |
| NFR-03 | [ADR-001](./work/20261011-image-analysis-agent/ADR-001-llm-role-split.md) | AC-07, AC-10 | TBD — 동상 |
| NFR-04 | [DES-07](./design.md#des-07-dlladapter), [ADR-002](./work/20261011-image-analysis-agent/ADR-002-dll-worker-process.md) | AC-05 | TBD — 동상 |
| NFR-05 | [9. 12시간 해커톤 진행 순서](./design.md#9-12시간-해커톤-진행-순서) | — | TBD — 리허설 |
| NFR-06 | [6. 품질 속성](./design.md#6-품질-속성) | — | TBD — 로그 검사 |
| NFR-07 | [DES-07](./design.md#des-07-dlladapter), [DES-08](./design.md#des-08-llmclient) | AC-04, AC-05, AC-13 | TBD — 동상 |
| NFR-08 | [DES-11](./design.md#des-11-webfrontend), [ADR-004](./work/20261011-image-analysis-agent/ADR-004-web-stack.md) | AC-11 | TBD — 오프라인 실행 확인 |

## 승인 기록

| 일자 | 승인자 | 대상 | 결과 | 기준선 | 근거 |
|---|---|---|---|---|---|
| 2026-10-11 | 사용자 | 1차 초안 | 수정 요청 | — | Q-01~Q-06 답변과 해커톤 제약 |
| 2026-10-11 | 사용자 | 2차 초안(MCP 도구 서버) | 수정 요청 | — | 2026-10-13 목표 변경(웹 개발) |
| 2026-10-13 | 사용자 | 3차 초안(이 문서)과 [설계](./design.md) | `awaiting-approval` | — | 대화에서 재승인 요청, 응답 대기 |

## 변경 이력

| 일자 | 변경 | 근거 | 상태 |
|---|---|---|---|
| 2026-10-11 | 최초 작성 | 사용자 요청 | `awaiting-approval` |
| 2026-10-11 | 2차 초안: 사용자 답변 반영, 해커톤 제약, MCP 도구 서버 구조, P0/P1/P2 | 사용자 1차 답변 | `awaiting-approval` |
| 2026-10-13 | 3차 초안: 목표를 웹 개발로 변경. FR-07·FR-09·FR-12 웹 기준 재정의, FR-13(수동 대체 경로)·NFR-08·C-06·AC-13~15·Q-11·RISK-09 추가, A-06/A-08(MCP 호스트) 폐기, A-10/A-11 추가, Q-07을 LLM API 질문으로 재정의 | 사용자 결정(2026-10-13) | `awaiting-approval` |
