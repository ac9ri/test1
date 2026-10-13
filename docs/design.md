# DESIGN-image-analysis-agent: 이미지 분석 AI Agent 설계

> 문서 유형: `design`
> 작업 ID: `20261011-image-analysis-agent`
> 상태: `awaiting-approval`
> 기준선: `draft`
> 작성일: 2026-10-11
> 최종 갱신: 2026-10-13
> 관련 문서: [REQ-image-analysis-agent: 요구사항](./requirements.md), [ADR-001](./work/20261011-image-analysis-agent/ADR-001-llm-role-split.md), [ADR-002](./work/20261011-image-analysis-agent/ADR-002-dll-worker-process.md), [ADR-003](./work/20261011-image-analysis-agent/ADR-003-llm-integration.md), [ADR-004](./work/20261011-image-analysis-agent/ADR-004-web-stack.md), [결정 등록부](./decisions.md), [웹 해커톤 프롬프트 팩](./hackathon-prompt-pack-web.md)

## 요약

- 목적: [요구사항](./requirements.md) 3차 초안(웹 개발)을 만족하는 구조·계약·동작을 정의하고, 사내 LLM API를 백엔드에서 쓰는 방법(5절)과 12시간 해커톤 진행 순서(9절)를 제시한다.
- 현재 결론 또는 상태: "FastAPI 백엔드 = Agent 오케스트레이터, 정적 HTML/JS 프런트엔드 = 사용자 확인 창구, 코어 = 결정적 분석" 구조를 제안한다. 백엔드가 작업(Job)을 상태 기계로 돌리며 불확실한 지점에서 `needs_input`으로 멈춰 브라우저에 확인을 요청한다. LLM은 OpenAI 호환 API로 판별 보조·서술·채팅에 쓰고, API가 없으면 수동 서술 경로로 동일 화면을 시연한다. 승인 대기.
- 다음 행동: 승인 후 기준선 `v1` 발행, wf-implement에서 9절 타임라인을 작업으로 분해한다.

## 문서 연결

| 방향 | 관계 | 대상 문서 | 대상 항목 | 비고 |
|---|---|---|---|---|
| input | baseline | [REQ-image-analysis-agent: 요구사항](./requirements.md) | FR-01~FR-13, NFR-01~NFR-08, AC-01~AC-15, C-01~C-06 | 입력 기준선 |
| input | decision | [ADR-001: LLM과 결정적 코드의 역할 분리](./work/20261011-image-analysis-agent/ADR-001-llm-role-split.md) | DES-02, DES-08, DES-12 | 판단 지점 최소화 |
| input | decision | [ADR-002: 과제 DLL 연동 방식](./work/20261011-image-analysis-agent/ADR-002-dll-worker-process.md) | DES-07 | ctypes 인프로세스 우선 |
| input | decision | [ADR-003: LLM 연동 방식 — 웹 백엔드의 OpenAI 호환 클라이언트와 수동 대체 경로](./work/20261011-image-analysis-agent/ADR-003-llm-integration.md) | DES-08, DES-10, 2.3, 5절 | 연동 모드 |
| input | decision | [ADR-004: 웹 기술 스택 선택](./work/20261011-image-analysis-agent/ADR-004-web-stack.md) | DES-10, DES-11, 8절 | FastAPI + 바닐라 JS |
| output | refinement | [DESIGN-image-analysis-agent-prompts-web: 웹 해커톤 프롬프트 팩](./hackathon-prompt-pack-web.md) | 3.3, 3.6, 5.3~5.6, 9.3 | 사내 LLM에 붙여넣을 빌드·런타임·운영 프롬프트 전문 |
| output | replacement | [DESIGN-image-analysis-agent-prompts: 해커톤 프롬프트 팩(MCP)](./hackathon-prompt-pack.md) | document | 목표 변경으로 대체됨(`superseded`) |
| output | related | [결정 등록부](./decisions.md) | document | ADR 목록 |

## 1. 설계 목표와 원칙

1. **코어는 LLM·웹과 무관하다(NFR-01).** 로더·기본 평가·DLL·판정·보고서는 순수 Python 패키지다. 웹 백엔드, CLI, LLM 클라이언트는 이를 감싸는 층이다.
2. **백엔드가 Agent 오케스트레이터다([ADR-003](./work/20261011-image-analysis-agent/ADR-003-llm-integration.md)).** 작업 상태 기계가 단계를 돌리고, LLM은 판별 보조·서술·채팅 세 지점에서만 호출된다. 불확실하면 멈추고 사람에게 묻는다.
3. **판단은 LLM, 계산은 코드([ADR-001](./work/20261011-image-analysis-agent/ADR-001-llm-role-split.md)).** 수치 표는 코드가 넣고 LLM은 표의 값만 인용한다.
4. **LLM 없이도 P0가 돈다.** API가 없거나 실패해도 결과 화면·판정·보고서가 나오고 서술만 수동으로 붙인다(FR-13).
5. **레지스트리 주도.** 차트·과제·DLL·임계값·LLM 설정은 YAML에 있다.
6. **해커톤 우선(C-01).** 프런트엔드는 화면 4개로 제한하고 꾸밈은 마지막에 한다. mock DLL·mock LLM으로 먼저 돌린다.

## 2. 시스템 구조

### 2.1 시스템 경계

```text
브라우저 (static/index.html + app.js, DES-11)
   │ HTTP/JSON, multipart 업로드, 1초 폴링
   ▼
FastAPI 백엔드 127.0.0.1:8000 (DES-10)
   ├─ JobManager: 작업 상태 기계, out/<job_id>/ 저장, needs_input 대기·재개
   ├─ Pipeline(DES-02) ─ ImageLoader+raw 추정(DES-03) ─ ChartClassifier(DES-04)
   │                   ─ BasicAnalyzer(DES-06) ─ DllAdapter ctypes/mock(DES-07) ─ Judge+Report(DES-09)
   ├─ LlmClient(DES-08, 선택) ──► 사내 LLM API (OpenAI 호환) : 판별 보조 · 서술 · 채팅
   ├─ ChatAgent(DES-12, P1): 도구 루프(get_result, run_analysis, get_catalog)
   └─ Registry(DES-05) ◄── config/registry.yaml
CLI(DES-01) ── 같은 코어, LLM 없이
```

### 2.2 컴포넌트

#### DES-01 CLI

- `analyze`, `validate-registry`, `catalog`, `attach-narrative` 명령. 코어를 직접 호출하고 LLM 없이 동작한다(AC-06). 계약은 [3.4](#34-cli-계약).

#### DES-02 Pipeline

- `run(path, chart_type=None, project_id=None, raw_override=None, registry, out_dir, llm=None, on_step=None) -> AnalysisResult`.
- 순서: 판별(규칙) → 과제 결정 → 로드(raw 메타 해결·추정) → 기본 평가 → DLL → 판정 → `result.json`·`preview.jpg` 저장.
- `llm`이 주어지고 판별이 `unknown`이면 프리뷰로 LLM 판별을 시도한다. 신뢰도가 `llm.confidence_threshold` 이상이면 채택하고 `method: llm`, 아니면 `unknown`을 유지하고 `candidates`를 채운다.
- 멈춤 조건은 예외가 아니라 결과로 표현한다. `classification.needs_classification` 또는 `input.raw_meta.needs_input`이 true면 JobManager가 `needs_input`으로 전이한다(FR-09). CLI는 그 상태를 `partial`로 출력하고 끝낸다.
- `on_step(stage, message)` 콜백으로 진행 로그를 Job에 쌓는다.

#### DES-03 ImageLoader와 raw 추정

- bmp/jpg: Pillow → 8비트 RGB. raw: 메타 해결 순서 = 요청 인자 > 사이드카 `<파일>.raw.json` > 레지스트리 `raw_defaults` > 추정(P1, [3.6](#36-raw-메타-추정-알고리즘)). 크기 검증 실패는 `E_RAW_SIZE_MISMATCH`.
- 디모자이크는 OpenCV. 패턴 상수 매핑은 반대 명명을 따른다: RGGB→`COLOR_BayerBG2RGB`, BGGR→`COLOR_BayerRG2RGB`, GRBG→`COLOR_BayerGB2RGB`, GBRG→`COLOR_BayerGR2RGB`.
- 프리뷰: 긴 변 1024px JPEG 품질 85. raw 프리뷰만 1~99 퍼센타일 스트레칭.

#### DES-04 ChartClassifier

- P0: 사용자 지정(`user`, 1.0) → `filename_regex`(`rule`, 1.0) → `unknown`(`none`, 0.0, `needs_classification: true`).
- P1: `unknown`일 때 `LlmClient.classify(preview, catalog)` → `ChartClassification(method="llm")`. 카탈로그 밖 ID는 `unknown`으로 강제.
- 사용자 선택 UI(DES-11)가 최종 결정권을 가진다.

#### DES-05 Registry

- YAML 로드·검증(pydantic), 조회, `catalog_dict()`, `llm` 설정 블록. 형식은 [3.2](#32-레지스트리-파일-형식). 실행 시작 전 검증 실패는 종료 코드 2 또는 서버 시작 실패.

#### DES-06 BasicAnalyzer

| 항목 ID | 산출 | 방법 |
|---|---|---|
| `basic.info` | 폭·높이·채널·비트 깊이·포맷 | ImageInfo |
| `basic.stats` | 채널별 mean/std/min/max, 휘도 평균 | numpy |
| `basic.clipping` | 포화 비율, 암부 비율 | numpy |
| `basic.sharpness` | 라플라시안 분산(중앙 50%×50% ROI, 전체) | OpenCV |
| `basic.noise` | 3×3 메디안 잔차의 MAD×1.4826 | numpy |
| `basic.uniformity` | 3×3 격자 평균, 코너/중앙 최소 비율 | numpy |
| `basic.bayer_stats` | raw 전용 R/Gr/Gb/B 평면 mean/std | numpy |

#### DES-07 DllAdapter

- P0: 인프로세스 `ctypes.WinDLL`([ADR-002](./work/20261011-image-analysis-agent/ADR-002-dll-worker-process.md)). 레지스트리 `dll.kind`: `c` | `mock`. 예외·반환 코드 ≠ 0은 `dll_status: failed`로 기록하고 계속(AC-05).
- 웹 서버에서 네이티브 크래시는 서버 종료로 이어지므로 P1 옵션으로 `pipeline.isolate: subprocess`를 두어 파이프라인을 `chart-agent analyze` 서브프로세스로 실행할 수 있게 한다. P2는 전용 워커.

#### DES-08 LlmClient

- 책임: 사내 LLM API 호출을 단일 인터페이스로 감싼다([ADR-003](./work/20261011-image-analysis-agent/ADR-003-llm-integration.md)). 구현체 `OpenAICompatibleClient`(운영), `MockLlmClient`(테스트·개발), `None`(미설정).

```python
class LlmClient(Protocol):
    def classify(self, preview_jpeg: bytes, filename: str, catalog: dict) -> ChartClassification: ...
    def narrative(self, tables_md: str, context: dict) -> Narrative: ...        # summary/interpretation/recommendation
    def chat(self, messages: list[dict], tools: list[dict]) -> ChatTurn: ...   # P1, 도구 호출 1스텝
    def probe(self) -> ProbeResult: ...                                        # 접속·이미지·json_schema·tools 지원 여부
```

- 구조화 출력 3단계 대체: `response_format: json_schema` → `json_object` → 프롬프트 내 스키마 + JSON 블록 추출. 모두 pydantic 검증 후 실패 시 오류를 붙여 1회 재요청.
- 설정: 환경 변수 `GLM_BASE_URL`, `GLM_API_KEY`, 레지스트리 `llm.model`, `llm.classify/narrative/chat` 파라미터. `GLM_BASE_URL`이 없으면 클라이언트는 `None`이고 모든 LLM 기능이 비활성화된다.
- 이미지는 Base64 Data URL. 서버 시작 시 `probe()` 결과를 `/api/health`에 노출한다.

#### DES-09 Judge와 ReportBuilder

- Judge: 임계값 문법 `>= x`, `> x`, `<= x`, `< x`, `== x`, `between a b`. 임계값 없는 항목은 `passed: null`.
- ReportBuilder: `report.md` = 머리말 + 기본 평가 표 + 과제 결과 표 + 판정 표 + 서술 3절 + 문제. 서술 출처를 `narrative_source: llm | template | manual`로 기록한다. 수치 정본은 `result.json`임을 머리에 명시한다.
- `interpret_prompt.md`(수동 경로용 해석 요청 프롬프트)도 같은 표로 생성한다.

#### DES-10 WebBackend

- FastAPI + uvicorn. 정적 파일(`static/`)을 `/`에 서빙하고 API는 `/api/*`. 계약은 [3.3](#33-rest-api-계약).
- JobManager: `jobs: dict[str, Job]` 인메모리 + `out/<job_id>/job.json` 영속. 작업은 `BackgroundTasks`로 실행한다(단일 프로세스, 순차 큐 1개). 서버 재시작 시 `out/`에서 목록 복구(P1).
- 상태 기계: `queued → running → done | partial | failed | needs_input`, `needs_input --resume--> running`. `partial`은 DLL 실패·LLM 미사용 등으로 결과는 있으나 완전하지 않은 상태다.
- 입력 검증: `chart_type`·`project_id`·`bayer`·`analysis_id`는 레지스트리 값만 허용하고 400과 허용 목록을 돌려준다(AC-09).
- 업로드: 확장자 화이트리스트, 200MB 상한, `out/<job_id>/input/<원본명>`에 저장.

#### DES-11 WebFrontend

- 빌드 도구·CDN 없는 `static/index.html`, `app.js`, `style.css`(NFR-08). 화면은 카드 4개로 제한한다.

| 카드 | 내용 | 우선순위 |
|---|---|---|
| 업로드 | 파일 선택·드롭, 차트 유형·과제 선택(카탈로그), raw 4개 필드, "LLM 사용" 체크, 분석 버튼 | P0 |
| 진행 | 1초 폴링, 단계 로그, 상태 배지 | P0 |
| 확인 요청 | `needs_input` 시 표시. chart_type: 프리뷰 + 후보 라디오(LLM 후보 우선, 카탈로그 전체 포함). raw_meta: 후보로 채운 폼 | P0 |
| 결과 | 프리뷰, 판정 표(합격 초록·불합격 빨강·참고 회색), 기본 평가·과제 결과 표, 보고서 본문(간단 렌더링), 다운로드, 서술 출처 배지, 수동 서술 영역(프롬프트 복사·붙여넣기·적용), 채팅(P1) | P0(채팅 제외) |

- 보고서 렌더링은 제목·표·목록만 처리하는 60줄 이내 변환 함수로 하고 실패하면 `<pre>`로 보여준다.
- 다중 파일(P1)은 업로드 카드에서 여러 파일을 받아 작업 목록 카드를 추가한다.

#### DES-12 ChatAgent

- P1. `POST /api/jobs/{id}/chat`. 시스템 프롬프트([5.5](#55-채팅-에이전트-프롬프트)) + 작업 결과 요약 + 사용자 메시지로 LLM 도구 루프를 최대 6스텝 돈다. 도구: `get_result()`, `run_analysis(analysis_id, roi)`, `get_catalog()`. 병렬 호출은 가정하지 않고 순차 처리한다. 응답에 도구 호출 내역을 포함해 화면에 표시한다(AC-15).

### 2.3 LLM 연동 모드

| 모드 | 조건 | 판별 보조 | 서술 | 채팅 | 시연 |
|---|---|---|---|---|---|
| auto | `GLM_BASE_URL` 설정 + probe 성공 | LLM 후보 → 임계값 미만이면 사용자 선택 | LLM 자동 | 가능(P1) | 완전 자동 + 확인 요청 |
| manual | API 없음 또는 probe 실패 | 카탈로그 후보 + 프리뷰로 사용자 선택 | 템플릿 → 수동 붙여넣기 | 불가 | 수치·판정 자동, 서술 수동 |
| off | 사용자가 "LLM 사용" 해제 | 사용자 선택 | 템플릿 | 불가 | 재현성 확인용 |

모드는 작업마다 `/api/jobs/{id}`의 `llm_mode`로 노출한다. 코어·백엔드·프런트엔드는 세 모드에서 같다.

### 2.4 의존 관계

- WebBackend → Pipeline, LlmClient, ChatAgent, Registry, ReportBuilder. CLI → Pipeline, Registry, ReportBuilder.
- Pipeline → Loader, Classifier, BasicAnalyzer, DllAdapter, Judge/Report, (선택) LlmClient.
- 프런트엔드는 REST API만 안다. 코어는 FastAPI·LLM을 import하지 않는다.

## 3. 데이터와 인터페이스

### 3.1 데이터 모델

```python
class RawMeta(BaseModel):
    width: int; height: int; bit_depth: int
    bayer: Literal["RGGB", "BGGR", "GRBG", "GBRG"]
    packing: Literal["unpacked16", "unpacked8"] = "unpacked16"
    estimated: bool = False; confidence: float = 1.0
    candidates: list[dict] = []; needs_input: bool = False

class RawOverride(BaseModel):
    width: int | None = None; height: int | None = None
    bit_depth: int | None = None; bayer: str | None = None

class ImageInfo(BaseModel):
    path: str; format: Literal["bmp", "jpg", "raw"]
    width: int; height: int; channels: int; bit_depth: int
    sha256: str; raw_meta: RawMeta | None = None

class Candidate(BaseModel):
    chart_type: str; confidence: float; reason: str = ""

class ChartClassification(BaseModel):
    chart_type: str; confidence: float
    method: Literal["user", "rule", "llm", "none"]
    needs_classification: bool = False
    candidates: list[Candidate] = []

class ProjectResult(BaseModel):
    project_id: str | None
    dll_status: Literal["ok", "failed", "skipped"]
    results: dict[str, float | int | str] = {}
    error: str | None = None; elapsed_ms: int | None = None

class Judgement(BaseModel):
    metric: str; value: float | int | str; rule: str | None; passed: bool | None

class Narrative(BaseModel):
    summary: str; interpretation: str; recommendation: str

class AnalysisResult(BaseModel):
    schema_version: Literal["1"] = "1"
    image_id: str; input: ImageInfo
    classification: ChartClassification
    basic: dict[str, dict]; project: ProjectResult
    judgement: list[Judgement]
    status: Literal["ok", "partial", "failed"]
    errors: list[dict] = []
    preview_path: str; report_path: str | None = None
    narrative_source: Literal["llm", "template", "manual"] | None = None

class Job(BaseModel):
    job_id: str; created_at: str; filename: str
    state: Literal["queued", "running", "needs_input", "done", "partial", "failed"]
    llm_mode: Literal["auto", "manual", "off"]
    steps: list[dict] = []                      # {"ts","stage","message","elapsed_ms"}
    needs: dict | None = None                   # {"type": "chart_type"|"raw_meta", "candidates": [...], "preview_url": ...}
    params: dict = {}                           # 마지막 요청 인자
    result: AnalysisResult | None = None
    error: str | None = None
```

### 3.2 레지스트리 파일 형식

```yaml
schema_version: 1
out_dir: ./out
server: {host: 127.0.0.1, port: 8000, max_upload_mb: 200}
llm:
  model: glm-5.3-flash
  base_url_env: GLM_BASE_URL
  api_key_env: GLM_API_KEY
  confidence_threshold: 0.7
  classify:  {temperature: 0.3, reasoning_effort: low,    timeout_s: 60,  max_tokens: 400,  max_image_edge: 1024}
  narrative: {temperature: 0.7, reasoning_effort: medium, timeout_s: 120, max_tokens: 1200}
  chat:      {temperature: 0.5, reasoning_effort: low,    timeout_s: 90,  max_tokens: 800,  max_steps: 6}
  retries: 2
pipeline: {isolate: none}      # none | subprocess (P1)
chart_types:
  - id: sfr_iso12233
    name: "ISO 12233 해상도(SFR) 차트"
    description_for_llm: "흰 배경에 검은 기울어진 사각형(slanted edge)이 여러 개 배치된 차트"
    filename_regex: "(?i)(sfr|mtf|res)"
    project: proj_sfr
  - id: flat_field
    name: "균일도(flat field) 차트"
    description_for_llm: "균일한 회색 또는 흰색 면"
    filename_regex: "(?i)(flat|uniform|gray)"
    project: null
projects:
  - id: proj_sfr
    name: "SFR 해상도 평가 과제"
    raw_defaults: {width: 4000, height: 3000, bit_depth: 10, bayer: RGGB}
    dll:
      kind: mock                       # c | mock
      path: "C:/tools/sfr/SfrCalc.dll"
      adapter: "chart_agent.dll.adapters.sfr:SfrAdapter"
      mock_adapter: "chart_agent.dll.adapters.mock_sfr:MockSfrAdapter"
    analyses:
      - id: proj_sfr.center
        function: CalcSFR
        params: {roi: center}
        thresholds: {mtf50: ">= 0.30", mtf30: ">= 0.45"}
basic_thresholds:
  basic.clipping.saturated_ratio: "<= 0.01"
  basic.sharpness.center: ">= 50"
```

### 3.3 REST API 계약

| 메서드·경로 | 요청 | 응답 | 우선순위 |
|---|---|---|---|
| `GET /api/health` | — | `{ok, llm: {configured, mode, model, probe}}` | P0 |
| `GET /api/catalog` | — | `registry.catalog_dict()` | P0 |
| `POST /api/jobs` | multipart `file`, 선택 `chart_type, project_id, width, height, bit_depth, bayer, use_llm` | 202 `{job_id}` | P0 |
| `GET /api/jobs/{id}` | — | `Job` JSON(`result` 포함, `basic.uniformity.grid` 등 큰 배열 생략 가능) | P0 |
| `POST /api/jobs/{id}/resume` | JSON `{chart_type?, width?, height?, bit_depth?, bayer?}` | 202 `{job_id}` | P0 |
| `GET /api/jobs/{id}/preview.jpg` | — | image/jpeg | P0 |
| `GET /api/jobs/{id}/report.md` | — | text/markdown | P0 |
| `GET /api/jobs/{id}/interpret-prompt` | — | text/plain(수동 경로 프롬프트) | P0 |
| `POST /api/jobs/{id}/narrative` | JSON `{summary, interpretation, recommendation}` 또는 `{markdown}` | `{report_path, narrative_source: "manual"}` | P0 |
| `GET /api/jobs` | — | `[Job 요약]` | P1 |
| `POST /api/jobs/{id}/chat` | JSON `{message}` | `{answer, tool_calls: [{name, args, result_summary}]}` | P1 |
| `POST /api/jobs/{id}/analyses/{analysis_id}` | JSON `{params}` | 해당 항목 결과 | P1 |

오류 응답은 `{error: {code, message, allowed?}}`. 코드는 [3.7](#37-오류-표현).

### 3.4 CLI 계약

```text
chart-agent analyze <파일|폴더> [--chart-type ID] [--project ID] [--width N --height N --bit-depth N --bayer RGGB] [--out DIR]
chart-agent validate-registry
chart-agent catalog
chart-agent attach-narrative <image_id> <narrative.md>
chart-agent serve [--host 127.0.0.1] [--port 8000]        # uvicorn 실행 래퍼
```

종료 코드 `0` 성공·부분 성공, `1` 예기치 않은 오류, `2` 입력·설정 오류.

### 3.5 DLL 어댑터 계약

```python
class DllAdapter(Protocol):
    def load(self, dll_path: str | None) -> None: ...
    def call(self, function: str, native: np.ndarray, info: ImageInfo, params: dict) -> dict[str, float | int | str]: ...
```

ctypes 예시와 mock 규칙은 2차 초안과 같다. 시그니처는 Q-08 확정 후 교체하며 mock은 중앙 ROI 라플라시안 분산 기반 결정적 pseudo-MTF를 반환한다.

### 3.6 raw 메타 추정 알고리즘

1. 바이트/픽셀 `b ∈ {2, 1}`에 대해 `N = S / b`가 정수인 경우만 후보.
2. `w × h == N`인 (w, h)를 레지스트리 `raw_defaults`와 기본 해상도 표에서 찾고, 없으면 `w % 8 == 0`이고 종횡비 4:3, 3:2, 16:9, 1:1(오차 2%)인 약수 쌍.
3. 각 후보를 `(h, w)`로 reshape해 행 `r`과 `r+2`의 평균 절대 차이(32행 간격 샘플)로 점수. 최저 점수 채택, `confidence = clamp(1 − best/second_best, 0, 1)`, 후보 하나면 0.9.
4. 비트 깊이는 최댓값이 들어가는 최소 비트 `{8,10,12,14,16}`.
5. 베이어: 4개 위상 평면 평균 중 가장 가까운 두 평면을 G로. R/B 순서는 기본값 우선, 없으면 둘 다 후보.
6. `confidence < 0.8` 또는 후보 2개 이상이면 `needs_input: true`.

### 3.7 오류 표현

`{stage, code, message}`. `stage ∈ {upload, load, classify, basic, dll, judge, report, llm, chat}`. `code ∈ {E_UNSUPPORTED_FORMAT, E_UPLOAD_TOO_LARGE, E_RAW_META_MISSING, E_RAW_SIZE_MISMATCH, E_RAW_ESTIMATE_AMBIGUOUS, E_DLL_LOAD, E_DLL_CALL, E_DLL_NOT_REGISTERED, E_UNKNOWN_ANALYSIS, E_UNKNOWN_CHART_TYPE, E_UNKNOWN_PROJECT, E_LLM_UNAVAILABLE, E_LLM_PARSE, E_JOB_NOT_FOUND, E_JOB_STATE}`.

## 4. 동작

### 4.1 정상 흐름(S-01, auto 모드)

1. 브라우저가 `POST /api/jobs`로 파일을 올린다. 백엔드는 `queued` 작업을 만들고 202로 `job_id`를 돌려준다.
2. BackgroundTask가 `running`으로 바꾸고 Pipeline을 돈다. 단계마다 `steps`에 로그를 쌓는다.
3. 규칙 판별 성공 → 로드 → 기본 평가 → DLL → 판정 → `result.json`·`preview.jpg`.
4. LLM 서술 호출 → `report.md`(`narrative_source: llm`) → `done`.
5. 브라우저는 1초마다 `GET /api/jobs/{id}`를 폴링하다 `done`이면 결과 카드를 그리고 `report.md`를 받는다.

### 4.2 확인 요청 흐름(S-03, S-04)

1. 판별이 `unknown`(LLM 후보가 있어도 임계값 미만)이거나 raw 메타가 불확실하면 Pipeline은 결과에 플래그를 남기고 종료한다.
2. JobManager가 `needs_input`으로 전이하고 `needs`에 유형·후보·프리뷰 URL을 넣는다.
3. 브라우저가 확인 카드를 그린다. 사용자가 선택·입력 후 `POST /api/jobs/{id}/resume`.
4. 백엔드는 `params`를 갱신하고 같은 `job_id`로 다시 `running`. 이전 단계 결과는 재계산한다(결정적이므로 동일).

### 4.3 예외 흐름

| 상황 | 백엔드 | 프런트엔드 |
|---|---|---|
| 업로드 형식·크기 오류 | 400 `E_UNSUPPORTED_FORMAT`/`E_UPLOAD_TOO_LARGE` | 업로드 카드에 메시지 |
| raw 크기 불일치 | `failed`, `error` | 진행 카드에 메시지, raw 폼 재표시 |
| DLL 실패 | `partial`, `dll_status: failed` | 결과 카드에 원인 배지 |
| LLM 미설정 | `llm_mode: manual`, 템플릿 서술, `done` 또는 `partial` | 수동 서술 영역 표시 |
| LLM 호출 실패·파싱 실패 | 재시도 후 템플릿 서술, `errors`에 `E_LLM_*` | 수동 서술 영역 + 경고 |
| 없는 `chart_type` 등 | 400 + `allowed` | 드롭다운 갱신 |
| 채팅 도구 오류 | 도구 결과로 오류·허용 목록 반환, LLM이 수정 | 도구 호출 내역 표시 |

### 4.4 타임아웃·재시도·동시성

- LLM: 호출별 타임아웃은 레지스트리 값. 네트워크·5xx·429만 지수 백오프 재시도, 4xx는 즉시 실패.
- DLL: 재시도 없음. `isolate: subprocess`(P1)에서만 타임아웃.
- 작업 큐는 1개(순차). 동시 업로드는 큐에 쌓인다. 시연에서는 3~5장으로 제한한다.
- 폴링 1초, 결과 JSON은 큰 배열을 빼서 50KB 이하.

## 5. GLM 연동 가이드: 백엔드의 OpenAI 호환 클라이언트

### 5.1 확인된 모델 특성

| 항목 | 내용 | 설계 영향 |
|---|---|---|
| 입력 | 텍스트·이미지(URL 또는 Base64 Data URL, 여러 장) | 프리뷰 Base64 |
| 함수 호출 | `tools`/`tool_choice` 지원, 병렬 호출은 미보장 | 채팅은 순차 루프 |
| 구조화 출력 | JSON 지원, JSON Schema 강제 여부 미명시 | 3단계 대체 |
| Thinking | 항상 켜짐, `reasoning_effort`로 조절 | 판별·채팅 `low`, 서술 `medium` |
| 권장 파라미터 | `temperature 1`, `top_p 0.95` | 판별은 0.3으로 낮춰 비교 |
| 컨텍스트·출력 | 1M / 128K | 넉넉하나 지연 때문에 줄인다 |

출처: [Z.ai GLM-5.3-Flash 문서](https://docs.z.ai/guides/vlm/glm-5.3-flash), [OpenRouter](https://openrouter.ai/z-ai/glm-5.3-flash) (2026-10-11 확인). 사내 API는 다를 수 있다(Q-07).

### 5.2 접속과 탐침

```python
from openai import OpenAI
client = OpenAI(base_url=os.environ["GLM_BASE_URL"], api_key=os.environ.get("GLM_API_KEY", "none"))
```

- 사내 CA가 있으면 `http_client=httpx.Client(verify="<CA 경로>")`.
- `probe()`는 P1 텍스트, P2 작은 이미지, P3 `json_schema`, P4 `json_object`, P5 프롬프트 JSON, P6 `tools` 1개, P7 `reasoning_effort` 지연을 순서대로 시도해 `ProbeResult`를 만든다. `/api/health`와 `scripts/probe_llm.py`가 같은 함수를 쓴다.
- GLM 고유 파라미터(`thinking`, `reasoning_effort`)는 `extra_body`로 넘기고 이름은 탐침으로 확인한다.

### 5.3 판별 호출

- system: 역할 + 카탈로그(`id`, `name`, `description_for_llm`) + 규칙(카탈로그 밖 ID 금지, 모르면 `unknown`, `confidence` 0~1, 후보 최대 3개) + 출력 스키마.
- user: 파일명·해상도 텍스트 + 프리뷰 이미지 1장(1024px JPEG).
- 응답 스키마: `{"chart_type": str, "confidence": float, "candidates": [{"chart_type", "confidence", "reason"}]}`. 검증 실패 시 오류를 붙여 1회 재요청, 재실패는 `unknown`.
- 전문은 [웹 프롬프트 팩 20](./hackathon-prompt-pack-web.md#20-런타임-판별-프롬프트).

### 5.4 서술 호출

- system: 역할 + 규칙(표의 값만 인용, 새 수치 금지, 한국어, 3절 고정) + 출력 스키마 `{"summary","interpretation","recommendation"}`.
- user: 과제 설명 + 차트 유형·판별 방법 + 기본 평가 표 + 과제 결과 표 + 판정 표(Markdown).
- P1: 응답 문장의 숫자를 추출해 표의 값 집합에 없으면 1회 재요청.
- 전문은 [웹 프롬프트 팩 21](./hackathon-prompt-pack-web.md#21-런타임-서술-프롬프트).

### 5.5 채팅 에이전트 프롬프트

- system: 역할 + 작업 요약(차트 유형·판정 요약) + 도구 사용 규칙(한 번에 하나, 수치는 도구 결과만, 등록되지 않은 분석은 거부) + 답변 형식(5줄 이내, 인용 수치 명시).
- 도구 스키마의 `analysis_id`는 `enum`으로 허용 ID만 넣는다.
- 전문은 [웹 프롬프트 팩 22](./hackathon-prompt-pack-web.md#22-런타임-채팅-에이전트-프롬프트와-도구-스키마).

### 5.6 수동 대체 경로

- `GET /api/jobs/{id}/interpret-prompt`가 [5.4](#54-서술-호출)의 system+user를 한 텍스트로 합쳐 준다. 사용자가 사내 채팅 프로그램에 붙여넣고, 답을 결과 카드의 영역에 붙여넣어 `POST /api/jobs/{id}/narrative`로 보낸다. 백엔드는 `## 요약`·`## 해석`·`## 권고` 절을 파싱하거나 JSON을 받아 보고서를 다시 만든다(`narrative_source: manual`).
- 전문은 [웹 프롬프트 팩 23](./hackathon-prompt-pack-web.md#23-수동-대체-경로-해석-요청-템플릿).

### 5.7 지연과 비용

- 판별 1회 + 서술 1회가 기본이다. 규칙 판별이 성공하면 서술 1회뿐이다. `reasoning_effort low`로 시작해 리허설에서 지연을 재고 `/api/jobs/{id}`의 `steps`에 LLM 소요 시간을 남긴다(NFR-06).
- 프리뷰는 1024px 1장만. 토큰은 `usage`를 로그에 기록한다.

## 6. 품질 속성

| 속성 | 설계 |
|---|---|
| 보안(NFR-02) | `127.0.0.1` 바인딩 기본, 업로드 화이트리스트·상한, 경로 조작 방지(파일명은 `job_id` 폴더 안으로만), 키는 환경 변수, 응답·로그에 키·이미지 바이트 없음. 인증 없음은 README에 명시 |
| 재현성(NFR-03) | 코어 LLM 무관, `result.json`에 레지스트리 해시·프롬프트 해시·모델 ID |
| 안정성(NFR-04) | DLL 예외 처리, `isolate: subprocess` 옵션(P1), uvicorn 재시작 안내 |
| 성능(NFR-05) | LLM 호출 ≤ 2회/장, 결과 JSON 소형화, 폴링 1초 |
| 관측성(NFR-06) | `out/<job_id>/job.json`의 `steps`, `server.log` |
| 유지보수성 | 레지스트리 주도, 어댑터 1파일, 프롬프트는 `prompts/*.md` |
| 테스트 가능성(NFR-07) | mock DLL, `MockLlmClient`(고정 응답·실패 시나리오), FastAPI `TestClient` |
| 오프라인(NFR-08) | 정적 파일만, 외부 CDN 없음 |

## 7. 검증 전략

| 인수 조건 | 전략 |
|---|---|
| AC-01, AC-02 | 합성 픽스처 단위 테스트 |
| AC-03 | 샘플 전부 CLI 판별 표 + 브라우저 확인 |
| AC-04, AC-05 | mock 정상·예외 테스트 |
| AC-06 | CLI 단독 실행 |
| AC-07 | 보고서 표 파싱 vs `result.json` |
| AC-08 | 테스트 레지스트리 |
| AC-09 | API `TestClient`로 400·`allowed` 확인 |
| AC-10 | 2회 실행 비교 |
| AC-11 | `TestClient`로 업로드→폴링→done→report 자동 테스트 + 브라우저 리허설 녹화 |
| AC-12 | 사양 아는 raw 추정 비교 |
| AC-13 | `GLM_BASE_URL` 없이 서버 기동 → interpret-prompt → narrative POST → `narrative_source: manual` |
| AC-14 | `unknown_01.jpg` 업로드 → `needs_input` → resume → done |
| AC-15 | `MockLlmClient`로 도구 호출 1회 시나리오 |

## 8. 기술 스택과 디렉터리 구조

- Python 3.11, `fastapi`, `uvicorn`, `python-multipart`, `openai`, `numpy`, `pillow`, `pydantic`, `pyyaml`, `opencv-python`, `pytest`, `httpx`(TestClient). 프런트엔드는 정적 HTML/CSS/JS.

```text
ai_agent/
├── docs/
├── config/registry.yaml
├── prompts/{classify.md, narrative.md, chat.md, interpret_template.md}
├── src/chart_agent/
│   ├── cli.py, pipeline.py, models.py, registry.py, errors.py, basic.py, classify.py, judge.py, report.py
│   ├── loaders/{__init__.py, image.py, raw.py, raw_estimate.py}
│   ├── dll/{base.py, __init__.py, adapters/sfr.py, adapters/mock_sfr.py}
│   ├── llm/{client.py, mock.py, probe.py, prompts.py}
│   └── web/{app.py, jobs.py, api.py, chat.py}
├── static/{index.html, app.js, style.css}
├── scripts/{check_dll.py, check_samples.py, probe_llm.py}
├── tests/
├── demo/
└── pyproject.toml
```

## 9. 12시간 해커톤 진행 순서

### 9.1 사전 준비

- 이 문서, [요구사항](./requirements.md), [웹 프롬프트 팩](./hackathon-prompt-pack-web.md) 반입. 샘플 확보(SFR 2장, raw 1장, 비차트 1장). DLL 시그니처(Q-08). 휠 다운로드. 사내 LLM API 정보(Q-07).

### 9.2 타임라인

| 시간 | 단계 | 산출물 | 빌드 프롬프트 |
|---|---|---|---|
| 0:00–0:30 | 환경 확인, DLL 로드 테스트, LLM API 탐침 | 확인 메모, `probe.json` | B0 |
| 0:30–1:30 | 모델·레지스트리·로더 | 테스트 통과 | B1 |
| 1:30–2:15 | 기본 평가·파이프라인·CLI | `result.json` | B2 |
| 2:15–3:30 | DLL 어댑터·mock·판정·보고서·수동 서술 | `report.md` | B3 |
| 3:30–4:30 | 웹 백엔드(작업 API, needs_input, 수동 경로) | `TestClient` 통과 | B4 |
| 4:30–6:00 | 프런트엔드 4카드 | 브라우저 업로드→보고서 | B5 |
| 6:00 | **P0 체크포인트(AC-11, AC-13, AC-14)** | — | — |
| 6:00–7:15 | LLM 클라이언트·판별·서술(API 있을 때) | auto 모드 | B6 |
| 7:15–8:30 | raw 추정, 채팅 에이전트 | S-03, S-07 | B7 |
| 8:30–9:30 | 다중 파일·작업 목록·오류 경로·로그·`isolate` 옵션 | S-02 | B8 |
| 9:30–10:30 | README·시연 데이터·안정화 | README | B9 |
| 10:30–11:30 | 리허설 2회, 지연 측정 | 기록 | — |
| 11:30–12:00 | 발표 자료 | 발표 | — |

B0에서 LLM API가 확인되면 B6를 B5 앞으로 당겨 자동 서술을 먼저 확보한다.

### 9.3 빌드 프롬프트

요지는 위 표와 같고, 사내 LLM이 이 문서를 못 읽어도 동작하도록 사양을 안에 담은 자기완결 전문은 [웹 프롬프트 팩](./hackathon-prompt-pack-web.md) 01~10에 있다.

### 9.4 체크포인트 규칙

- 6:00에 브라우저에서 업로드→보고서(수동 서술 포함)가 안 되면 P1을 모두 중단한다.
- 실제 DLL이 로드되지 않으면 mock으로 시연한다.
- LLM API가 없으면 manual 모드로 시연하고 발표에서 "API 연결 시 자동" 구조를 설명한다.

## 추적성

| 요구사항 | 설계 요소 | 인수 조건 | 검증 |
|---|---|---|---|
| [FR-01](./requirements.md#4-기능-요구사항) | DES-03 | AC-01, AC-02 | TBD — wf-implement에서 VER 발행 |
| [FR-02](./requirements.md#4-기능-요구사항) | DES-04, DES-08, 5.3 | AC-03, AC-14 | TBD — 동상 |
| [FR-03](./requirements.md#4-기능-요구사항) | DES-05, DES-10 검증 | AC-09 | TBD — 동상 |
| [FR-04](./requirements.md#4-기능-요구사항) | DES-06 | AC-01, AC-10 | TBD — 동상 |
| [FR-05](./requirements.md#4-기능-요구사항) | DES-07, 3.5 | AC-04, AC-05 | TBD — 동상 |
| [FR-06](./requirements.md#4-기능-요구사항) | DES-09, DES-08, 5.4 | AC-06, AC-07 | TBD — 동상 |
| [FR-07](./requirements.md#4-기능-요구사항) | DES-01, DES-10, DES-11, 3.3, 3.4 | AC-06, AC-11 | TBD — 동상 |
| [FR-08](./requirements.md#4-기능-요구사항) | DES-05, 3.2 | AC-08 | TBD — 동상 |
| [FR-09](./requirements.md#4-기능-요구사항) | DES-02, DES-10 상태 기계, DES-11 확인 카드, 4.2 | AC-14 | TBD — 동상 |
| [FR-10](./requirements.md#4-기능-요구사항) | DES-03, 3.6 | AC-12 | TBD — 동상 |
| [FR-11](./requirements.md#4-기능-요구사항) | DES-10 `GET /api/jobs`, DES-11 | — | TBD — 리허설 |
| [FR-12](./requirements.md#4-기능-요구사항) | DES-12, 5.5 | AC-15 | TBD — 동상 |
| [FR-13](./requirements.md#4-기능-요구사항) | DES-09 `interpret_prompt`, DES-10 `narrative`, DES-11, 5.6 | AC-13 | TBD — 동상 |
| [NFR-01](./requirements.md#5-비기능-요구사항) | 1절 원칙 1, 2.4 | AC-06, AC-11 | TBD — 동상 |
| [NFR-02](./requirements.md#5-비기능-요구사항) | 6절 보안 | — | TBD — 코드 리뷰 |
| [NFR-03](./requirements.md#5-비기능-요구사항) | ADR-001, DES-09 | AC-07, AC-10 | TBD — 동상 |
| [NFR-04](./requirements.md#5-비기능-요구사항) | DES-07, ADR-002 | AC-05 | TBD — 동상 |
| [NFR-05](./requirements.md#5-비기능-요구사항) | 4.4, 5.7, 9.2 | — | TBD — 리허설 |
| [NFR-06](./requirements.md#5-비기능-요구사항) | 6절 관측성 | — | TBD — 로그 검사 |
| [NFR-07](./requirements.md#5-비기능-요구사항) | DES-07 mock, DES-08 mock, 7절 | AC-04, AC-05, AC-13, AC-15 | TBD — 동상 |
| [NFR-08](./requirements.md#5-비기능-요구사항) | DES-11, ADR-004 | AC-11 | TBD — 오프라인 확인 |

## 승인 기록

| 일자 | 승인자 | 대상 | 결과 | 기준선 | 근거 |
|---|---|---|---|---|---|
| 2026-10-11 | 사용자 | 1차 초안 | 수정 요청 | — | Q-01~Q-06 답변 |
| 2026-10-11 | 사용자 | 2차 초안(MCP) | 수정 요청 | — | 2026-10-13 목표 변경 |
| 2026-10-13 | 사용자 | 3차 초안(이 문서), [요구사항](./requirements.md), ADR-001~004 | `awaiting-approval` | — | 재승인 요청, 응답 대기 |

## 인계

- 다음 단계 또는 워크플로우: 승인 후 [wf-implement](file:///C:/Users/JS/.claude/skills/wf-implement/SKILL.md) 계획 수립. 9.2 타임라인을 TASK로 분해한다.
- 시작 조건: 기준선 `v1`, ADR-001~004 `approved`, Q-07 답변(없으면 manual 모드 가정)
- 입력 문서와 기준선: [REQ-image-analysis-agent](./requirements.md), 이 문서, [ADR-001](./work/20261011-image-analysis-agent/ADR-001-llm-role-split.md), [ADR-002](./work/20261011-image-analysis-agent/ADR-002-dll-worker-process.md), [ADR-003](./work/20261011-image-analysis-agent/ADR-003-llm-integration.md), [ADR-004](./work/20261011-image-analysis-agent/ADR-004-web-stack.md), [웹 프롬프트 팩](./hackathon-prompt-pack-web.md)
- 완료된 항목: 요구사항·설계 3차 초안, ADR 4건, 웹 프롬프트 팩
- 미완료 항목: 사용자 승인, Q-07~Q-11 답변, 기준선 발행
- 차단 요인: 승인 전 구현 시작 불가
- 다음 행동: 사용자 응답 반영 후 상태 갱신

## 변경 이력

| 일자 | 변경 | 근거 | 상태 |
|---|---|---|---|
| 2026-10-11 | 최초 작성(API 클라이언트 기반 Agent) | 요구사항 1차 초안 | `awaiting-approval` |
| 2026-10-11 | 2차 초안: 사내 호스트 프로그램 + MCP 도구 서버 | 사용자 1차 답변 | `awaiting-approval` |
| 2026-10-12 | MCP 프롬프트 팩 부속 추가 | 사용자 요청 | `awaiting-approval` |
| 2026-10-13 | 3차 초안: 웹 앱 구조로 전환. DES-10 WebBackend·DES-11 WebFrontend·DES-12 ChatAgent 추가, DES-08을 LlmClient로 복원, MCP 서버 제거, LLM 연동 모드(auto/manual/off), REST API 계약, Job 모델, 수동 서술 경로, 타임라인 B0~B9 재편, 웹 프롬프트 팩 링크, MCP 팩은 `superseded` | 사용자 결정(2026-10-13) | `awaiting-approval` |
