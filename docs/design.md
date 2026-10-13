# DESIGN-image-analysis-agent: 이미지 분석 AI Agent 설계

> 문서 유형: `design`
> 작업 ID: `20261011-image-analysis-agent`
> 상태: `awaiting-approval`
> 기준선: `draft`
> 작성일: 2026-10-11
> 최종 갱신: 2026-10-11
> 관련 문서: [REQ-image-analysis-agent: 요구사항](./requirements.md), [ADR-001](./work/20261011-image-analysis-agent/ADR-001-llm-role-split.md), [ADR-002](./work/20261011-image-analysis-agent/ADR-002-dll-worker-process.md), [ADR-003](./work/20261011-image-analysis-agent/ADR-003-llm-host-mcp-integration.md), [결정 등록부](./decisions.md)

## 요약

- 목적: [요구사항](./requirements.md) 2차 초안을 만족하는 구조·계약·동작을 정의하고, 사내 호스트 프로그램(클로드 데스크톱 유사)에서 GLM-5.3-Flash로 Agent를 돌리기 위한 프롬프트 전문(5절)과 12시간 해커톤 진행 순서·빌드 프롬프트(9절)를 제시한다.
- 현재 결론 또는 상태: "사내 호스트 프로그램 = Agent 런타임, 우리 코드 = MCP 도구 서버 + 같은 코어를 쓰는 CLI" 구조를 제안한다. 매크로 도구 `analyze_image` 하나가 로드→판별→기본 평가→DLL→판정을 한 번에 처리해 LLM 호출 수를 이미지당 2~3회로 줄인다. 승인 대기.
- 다음 행동: 승인 후 기준선 `v1` 발행, wf-implement에서 9절 타임라인을 작업으로 분해한다.

## 문서 연결

| 방향 | 관계 | 대상 문서 | 대상 항목 | 비고 |
|---|---|---|---|---|
| input | baseline | [REQ-image-analysis-agent: 요구사항](./requirements.md) | FR-01~FR-12, NFR-01~NFR-07, AC-01~AC-12, C-01~C-05 | 이 설계의 입력 기준선 |
| input | decision | [ADR-001: LLM과 결정적 코드의 역할 분리](./work/20261011-image-analysis-agent/ADR-001-llm-role-split.md) | DES-02, DES-08, DES-10 | 매크로 도구·수치 인용 규칙의 근거 |
| input | decision | [ADR-002: 과제 DLL 연동 방식](./work/20261011-image-analysis-agent/ADR-002-dll-worker-process.md) | DES-07 | ctypes 인프로세스 우선 |
| input | decision | [ADR-003: LLM 연동 방식 — 사내 호스트 프로그램 + MCP 도구 서버](./work/20261011-image-analysis-agent/ADR-003-llm-host-mcp-integration.md) | DES-08, DES-10, 2.3 | 실행 주체와 연동 사다리 |
| output | refinement | [DESIGN-image-analysis-agent-prompts: 해커톤 프롬프트 팩](./hackathon-prompt-pack.md) | DES-10, 5.3, 5.4, 9.3 | 사내 LLM에 붙여넣을 빌드·런타임·운영 프롬프트 전문 |
| output | related | [결정 등록부](./decisions.md) | document | ADR 목록 |

## 1. 설계 목표와 원칙

1. **코어는 LLM·호스트와 무관하다(NFR-01).** 로더·기본 평가·DLL·판정·보고서는 순수 Python 패키지다. MCP 서버와 CLI는 이를 감싸는 얇은 층이다. 호스트가 바뀌어도 코어는 그대로다.
2. **호스트가 Agent 런타임이다([ADR-003](./work/20261011-image-analysis-agent/ADR-003-llm-host-mcp-integration.md)).** GLM의 도구 호출 루프는 호스트가 제공한다. 우리는 도구와 시스템 프롬프트만 만든다. LLM 클라이언트·재시도·구조화 출력 파싱 코드를 쓰지 않는다.
3. **판단은 LLM, 계산은 코드([ADR-001](./work/20261011-image-analysis-agent/ADR-001-llm-role-split.md)).** 매크로 도구가 결정적 단계를 묶어 처리하고, LLM은 판별 보조·사용자 확인·해석 문장만 맡는다.
4. **레지스트리 주도.** 차트 유형·과제·DLL·임계값은 YAML에 선언한다. 시스템 프롬프트의 카탈로그는 `get_chart_catalog` 도구로 레지스트리에서 읽는다.
5. **해커톤 우선(C-01).** P0 경로를 6시간 30분 안에 끝낸다. 실제 DLL이 늦어도 진행되도록 mock을 같은 시그니처로 먼저 만든다(NFR-07).

## 2. 시스템 구조

### 2.1 시스템 경계

```text
┌─ 사내 호스트 프로그램 (GLM-5.3-Flash) ───────────────────────────┐
│  시스템 프롬프트(DES-10)  ·  도구 호출 루프  ·  사용자 채팅        │
└───────────────┬───────────────────────────────────────────────┘
                │ MCP (stdio)  ── 도구: list_inputs, get_chart_catalog,
                ▼                 analyze_image, get_preview, write_report
┌─ chart_agent MCP 서버 (DES-08) ────────────────────────────────┐
│                                                                │
│  ┌─ 코어 패키지 ───────────────────────────────────────────┐  │
│  │ Pipeline(DES-02)                                         │  │
│  │  ├ ImageLoader + raw 추정(DES-03)                        │  │
│  │  ├ ChartClassifier 규칙(DES-04)                          │  │
│  │  ├ BasicAnalyzer(DES-06)                                 │  │
│  │  ├ DllAdapter ctypes(DES-07) ──► 과제 C DLL / mock       │  │
│  │  └ Judge + ReportBuilder(DES-09)                         │  │
│  │ Registry(DES-05) ◄── config/registry.yaml                │  │
│  └──────────────────────────────────────────────────────────┘  │
│         ▲                                                      │
│  CLI(DES-01) ── 같은 코어, LLM 없이 result.json + 템플릿 보고서  │
└────────────────────────────────────────────────────────────────┘
        │ 출력: out/<image_id>/result.json, preview.jpg, report.md, run.log
```

### 2.2 컴포넌트

#### DES-01 CLI

- 책임: `analyze`, `validate-registry`, `attach-narrative`(채팅 전용 대체 경로용) 명령. 코어를 직접 호출하고 LLM 없이 동작한다(AC-06).
- 계약은 [3.4 CLI 계약](#34-cli-계약).

#### DES-02 Pipeline

- 책임: `run(path, chart_type=None, project_id=None, raw_override=None) -> AnalysisResult`. 로드 → 판별 → 기본 평가 → 과제 DLL → 판정을 순서대로 실행하고 단계별 예외를 `errors[]`로 모은다. 상태는 `ok`/`partial`/`failed`.
- `image_id`: 입력 파일 sha256 앞 12자리. 출력 폴더 `out/<image_id>/`에 `result.json`, `preview.jpg`를 쓴다. 같은 파일을 다시 분석하면 덮어쓴다(NFR-03).
- 폴더 입력(P1)은 CLI와 `list_inputs` 도구가 파일 목록만 만들고, 파일마다 `run`을 호출한다.

#### DES-03 ImageLoader와 raw 추정

- bmp/jpg: Pillow 로드 → 8비트 RGB 배열. 그레이스케일이면 채널 1.
- raw: 메타 결정 순서 = 호출 인자 > 사이드카 `<파일>.raw.json` > 레지스트리 `projects[*].raw_defaults`(project_id가 있을 때) > 추정([3.6](#36-raw-메타-추정-알고리즘), P1). 결정된 메타로 `파일 크기 == 폭 × 높이 × 바이트`를 검증하고 불일치면 `E_RAW_SIZE_MISMATCH`.
- 디모자이크: OpenCV `cv2.cvtColor(bayer, COLOR_BAYER_xx2RGB)` → 비트 깊이 정규화 → 8비트 프리뷰. 프리뷰는 긴 변 1024px JPEG(품질 85)로 저장한다. 어두운 raw는 프리뷰에만 1~99 퍼센타일 스트레칭을 적용한다.
- 출력: full-res 배열(원본 비트 깊이, DLL·수치용), 프리뷰, `ImageInfo`.

#### DES-04 ChartClassifier

- P0: 사용자 지정(`method: user`, 신뢰도 1.0) → 레지스트리 `filename_regex`(`method: rule`, 1.0) → `unknown`(`method: none`, 0.0).
- P1: `unknown`일 때 도구 결과에 `preview_path`와 `needs_classification: true`를 넣는다. 호스트 LLM이 `get_preview`로 보고 유형을 정해 `chart_type`을 지정해 재호출한다(DES-10 절차 3). 즉 VLM 판별은 코드가 아니라 프롬프트로 구현한다.
- P2: ArUco 마커, 템플릿 매칭.

#### DES-05 Registry

- `config/registry.yaml`을 pydantic으로 로드·검증한다. 조회: 차트 유형 → 과제 → 분석 항목 → DLL 함수 → 임계값. `catalog_text()`로 LLM용 카탈로그 문자열을 만든다.
- 형식은 [3.2](#32-레지스트리-파일-형식).

#### DES-06 BasicAnalyzer

| 항목 ID | 산출 | 방법 |
|---|---|---|
| `basic.info` | 폭·높이·채널·비트 깊이·포맷 | ImageInfo |
| `basic.stats` | 채널별 mean/std/min/max, 휘도 평균 | numpy |
| `basic.clipping` | 포화 비율, 암부 비율 | numpy |
| `basic.sharpness` | 라플라시안 분산(중앙 25% ROI, 전체) | OpenCV |
| `basic.noise` | 3×3 메디안 잔차의 MAD×1.4826 | numpy |
| `basic.uniformity` | 3×3 격자 평균 휘도, 코너/중앙 최소 비율 | numpy |
| `basic.bayer_stats` | raw 전용: R/Gr/Gb/B 평면 mean/std | numpy |

모두 결정적이다. 히스토그램은 `result.json` 크기를 키우므로 P1에서 16-bin 요약만 넣는다.

#### DES-07 DllAdapter

- P0: 메인 프로세스에서 `ctypes.WinDLL`로 로드하고 호출한다([ADR-002](./work/20261011-image-analysis-agent/ADR-002-dll-worker-process.md)). 예외·비정상 반환 코드는 `dll_status: failed`로 기록하고 파이프라인은 계속한다(AC-05).
- 레지스트리 `dll.kind`: `c`(ctypes) 또는 `mock`(같은 시그니처의 Python 함수). mock은 실제 DLL이 없을 때와 테스트에 쓴다(NFR-07).
- 어댑터는 과제당 1파일(`dll/adapters/sfr.py`)이며 [3.5 계약](#35-dll-어댑터-계약)을 구현한다. 시그니처는 Q-08 확정 후 교체한다.
- P2: 별도 워커 프로세스 격리(원 설계 유지, 32비트 DLL이면 앞당김).

#### DES-08 McpToolServer

- `mcp` 패키지의 FastMCP로 stdio 서버를 띄운다. 도구 5개(P0·P1)와 선택 도구 1개(P2)를 노출한다. 계약은 [3.3](#33-mcp-도구-계약).
- 도구는 코어를 호출하고 결과를 JSON 문자열로 반환한다. `get_preview`만 이미지 콘텐츠를 반환한다.
- 입력 검증: `chart_type`·`project_id`·`analysis_id`는 레지스트리에 있는 값만 허용하고, 아니면 허용 목록을 담은 오류를 반환한다(AC-09).
- 환경 변수 `CHART_AGENT_REGISTRY`, `CHART_AGENT_OUT`로 경로를 받는다.

#### DES-09 Judge와 ReportBuilder

- Judge: 레지스트리 `thresholds`(`>= 0.3`, `between 0.4 0.6` 문법)와 `project.results`·`basic`을 비교해 `judgement[]`를 만든다. 임계값이 없는 항목은 `passed: null`(참고용).
- ReportBuilder: `report.md` = 머리말(파일·유형·판별 방법·상태) + 코드가 렌더링한 기본 평가 표·과제 결과 표·판정 표 + 서술 3절(요약·해석·권고). 서술은 `write_report` 인자로 호스트 LLM이 주며, 없으면 템플릿 문장("판정 표를 참조하라. 불합격 N건.")을 쓴다. 보고서 머리에 "수치 정본은 result.json"을 명시한다.

#### DES-10 HostAgentPrompt

- 책임: 호스트 프로그램에 넣는 시스템 프롬프트(프로젝트 지침)와 도구 설명문. 코드가 아니라 텍스트 산출물이지만 Agent 동작을 결정하는 설계 요소다. 전문은 [5.3](#53-호스트-시스템-프롬프트-전문)과 [5.4](#54-도구-설명문). 대체 경로·문제 해결 프롬프트까지 포함한 붙여넣기용 전체본은 [해커톤 프롬프트 팩](./hackathon-prompt-pack.md)이다.
- 절차(판별 → 확인 → 보고서), 수치 인용 규칙, `unknown`·`needs_input` 처리, 한 번에 하나의 도구 호출을 지시한다.

### 2.3 연동 사다리

호스트의 확장 방식(Q-07)에 따라 아래 순서로 내려간다. 코어와 CLI는 모든 단계에서 같다.

| 단계 | 호스트 능력 | 우리가 만드는 것 | LLM이 하는 일 |
|---|---|---|---|
| A (P0 가정) | MCP 서버 등록·도구 호출 | MCP 서버 + 시스템 프롬프트 | 도구 호출, 프리뷰 판별, 해석 |
| B | 로컬 명령 실행·파일 읽기(코워크형) | CLI + 시스템 프롬프트(도구 대신 명령 사용법 기술) | CLI 실행, `result.json`·`preview.jpg` 읽기, `narrative.md` 작성 |
| C | 채팅만 | CLI가 `interpret_prompt.md` 생성, `attach-narrative` 명령 | 사용자가 붙여넣은 표를 보고 3절 작성 |
| D | 별도 API 존재 | 원 설계의 `OpenAICompatibleClient`를 CLI에 추가 | A와 동일, 호스트 없이 자동 |

### 2.4 의존 관계

- MCP 서버와 CLI → 코어. 코어는 둘을 모르고 LLM도 모른다.
- Pipeline → Loader, Classifier, BasicAnalyzer, DllAdapter, Judge/Report, Registry.
- DllAdapter는 Registry의 DLL 설정만 의존한다.

## 3. 데이터와 인터페이스

### 3.1 데이터 모델

```python
class RawMeta(BaseModel):
    width: int; height: int; bit_depth: int
    bayer: Literal["RGGB", "BGGR", "GRBG", "GBRG"]
    packing: Literal["unpacked16", "unpacked8"] = "unpacked16"
    estimated: bool = False
    confidence: float = 1.0
    candidates: list[dict] = []          # 추정 후보 {width,height,bit_depth,bayer,score}
    needs_input: bool = False            # 신뢰도 < 0.8 또는 후보 다수

class ImageInfo(BaseModel):
    path: str; format: Literal["bmp", "jpg", "raw"]
    width: int; height: int; channels: int; bit_depth: int
    sha256: str; raw_meta: RawMeta | None = None

class ChartClassification(BaseModel):
    chart_type: str                      # 레지스트리 ID 또는 "unknown"
    confidence: float
    method: Literal["user", "rule", "none"]
    needs_classification: bool = False   # unknown이면 true

class ProjectResult(BaseModel):
    project_id: str | None
    dll_status: Literal["ok", "failed", "skipped"]
    results: dict[str, float | int | str] = {}
    error: str | None = None
    elapsed_ms: int | None = None

class Judgement(BaseModel):
    metric: str; value: float | int | str; rule: str | None; passed: bool | None

class AnalysisResult(BaseModel):
    schema_version: Literal["1"] = "1"
    image_id: str
    input: ImageInfo
    classification: ChartClassification
    basic: dict[str, dict]
    project: ProjectResult
    judgement: list[Judgement]
    status: Literal["ok", "partial", "failed"]
    errors: list[dict] = []              # {"stage","code","message"}
    preview_path: str
    report_path: str | None = None
```

상태 규칙: 모든 단계 성공 `ok`. `unknown`, DLL 실패, raw 추정 사용 중 하나라도 있으면 `partial`. 로드 실패 `failed`.

### 3.2 레지스트리 파일 형식

```yaml
schema_version: 1
out_dir: ./out

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
    raw_defaults: {width: 4000, height: 3000, bit_depth: 10, bayer: RGGB}   # Q-09 확정 후 교체
    dll:
      kind: c                     # c | mock
      path: "C:/tools/sfr/SfrCalc.dll"
      adapter: "chart_agent.dll.adapters.sfr:SfrAdapter"
    analyses:
      - id: proj_sfr.center
        function: CalcSFR         # Q-08 확정 후 교체
        params: {roi: center}
        thresholds: {mtf50: ">= 0.30", mtf30: ">= 0.45"}

basic_thresholds:                 # 전체 이미지 평가 판정(선택)
  basic.clipping.saturated_ratio: "<= 0.01"
  basic.sharpness.center: ">= 50"
```

### 3.3 MCP 도구 계약

| 도구 | 인자 | 반환 | 우선순위 |
|---|---|---|---|
| `list_inputs` | `path: str` | JSON `[{path, ext, size_bytes, has_sidecar}]` | P1 |
| `get_chart_catalog` | 없음 | JSON 차트 유형·과제·분석 항목·임계값 | P0 |
| `analyze_image` | `path`, `chart_type?`, `project_id?`, `width?`, `height?`, `bit_depth?`, `bayer?` | JSON `AnalysisResult` | P0 |
| `get_preview` | `image_id` | 이미지(JPEG, 1024px) | P1 |
| `write_report` | `image_id`, `summary`, `interpretation`, `recommendation` | JSON `{report_path, status}` | P0 |
| `run_analysis` | `image_id`, `analysis_id`, `params?` | JSON 해당 항목 결과 | P2 |

```python
from mcp.server.fastmcp import FastMCP, Image
mcp = FastMCP("chart-agent")

@mcp.tool()
def analyze_image(path: str, chart_type: str | None = None, project_id: str | None = None,
                  width: int | None = None, height: int | None = None,
                  bit_depth: int | None = None, bayer: str | None = None) -> str:
    """(5.4의 analyze_image 설명문을 그대로 넣는다)"""
    result = pipeline.run(path, chart_type, project_id,
                          raw_override=RawOverride(width, height, bit_depth, bayer))
    return result.model_dump_json(indent=1, exclude={"basic": {"histogram"}})

@mcp.tool()
def get_preview(image_id: str) -> Image:
    """(5.4의 get_preview 설명문)"""
    return Image(path=str(store.preview_path(image_id)))

if __name__ == "__main__":
    mcp.run()   # stdio
```

반환 JSON은 2~4KB 안에 유지한다. thinking 모델이 긴 도구 결과를 읽는 시간이 지연으로 직결된다.

### 3.4 CLI 계약

```text
chart-agent analyze <파일|폴더> [--chart-type ID] [--project ID]
                    [--width N --height N --bit-depth N --bayer RGGB] [--out DIR]
chart-agent validate-registry
chart-agent attach-narrative <image_id> <narrative.md>     # 연동 사다리 C용
```

종료 코드 `0` 성공·부분 성공, `1` 예기치 않은 오류, `2` 입력·설정 오류. 폴더 입력은 `out/summary.json`에 파일별 상태·판정 요약을 쓴다.

### 3.5 DLL 어댑터 계약

```python
class DllAdapter(Protocol):
    def load(self, dll_path: str | None) -> None: ...          # kind: mock이면 무시
    def call(self, function: str, image: np.ndarray, info: ImageInfo,
             params: dict) -> dict[str, float | int | str]: ...
```

ctypes 예시(시그니처는 Q-08 확정 후 교체):

```python
class SfrAdapter:
    def load(self, dll_path):
        self.lib = ctypes.WinDLL(dll_path)
        f = self.lib.CalcSFR
        f.argtypes = [ctypes.POINTER(ctypes.c_uint16), ctypes.c_int, ctypes.c_int,
                      ctypes.c_int, ctypes.c_int, ctypes.c_int, ctypes.c_int,
                      ctypes.POINTER(ctypes.c_double)]
        f.restype = ctypes.c_int
    def call(self, function, image, info, params):
        buf = np.ascontiguousarray(image, dtype=np.uint16)
        x, y, w, h = roi_from(params, info)
        out = (ctypes.c_double * 2)()
        rc = getattr(self.lib, function)(buf.ctypes.data_as(ctypes.POINTER(ctypes.c_uint16)),
                                         info.width, info.height, x, y, w, h, out)
        if rc != 0:
            raise RuntimeError(f"{function} rc={rc}")
        return {"mtf50": out[0], "mtf30": out[1]}
```

mock(`kind: mock`)은 같은 `call` 시그니처로 중앙 ROI의 라플라시안 분산에서 결정적 pseudo-MTF를 계산해 반환한다. 실제 DLL이 오면 어댑터 1파일과 레지스트리 `kind`만 바꾼다.

### 3.6 raw 메타 추정 알고리즘

입력: 파일 크기 `S`, 레지스트리의 알려진 해상도 목록, 기본 해상도 표(640×480, 1280×720, 1920×1080, 2592×1944, 3264×2448, 4000×3000, 4032×3024, 4608×3456, 8000×6000 등).

1. 바이트/픽셀 `b ∈ {2, 1}`에 대해 `N = S / b`가 정수인 경우만 남긴다.
2. `w × h == N`인 (w, h)를 해상도 표에서 찾는다. 없으면 `w % 8 == 0`이고 종횡비가 4:3, 3:2, 16:9, 1:1 중 하나(오차 2%)인 약수 쌍을 후보로 만든다.
3. 각 후보를 `(h, w)`로 reshape하고 행 `r`와 `r+2`(같은 베이어 위상)의 평균 절대 차이를 32행 간격으로 샘플링해 점수로 삼는다. 점수가 가장 낮은 후보를 고르고, `confidence = clamp(1 − best / second_best, 0, 1)`로 둔다. 후보가 하나면 0.9.
4. 비트 깊이: 최댓값이 들어가는 최소 비트를 `{8, 10, 12, 14, 16}`에서 고른다. `b == 2`인데 최댓값 ≤ 255이면 8비트 저장 의심으로 후보에 둘 다 넣는다.
5. 베이어: 4개 위상 평면의 평균 중 가장 가까운 두 평면을 G로 본다. G가 (0,1)·(1,0)이면 `RGGB|BGGR`, (0,0)·(1,1)이면 `GRBG|GBRG`. R/B 순서는 레지스트리 기본값이 있으면 그것을, 없으면 두 후보를 모두 반환한다(AC-12에서 R/B 순서는 후보 허용).
6. `confidence < 0.8` 또는 후보 2개 이상이면 `needs_input: true`.

### 3.7 오류 표현

`errors[]` 항목 `{stage, code, message}`. `stage ∈ {load, classify, basic, dll, judge, report}`, `code ∈ {E_UNSUPPORTED_FORMAT, E_RAW_META_MISSING, E_RAW_SIZE_MISMATCH, E_RAW_ESTIMATE_AMBIGUOUS, E_DLL_LOAD, E_DLL_CALL, E_DLL_NOT_REGISTERED, E_UNKNOWN_ANALYSIS, E_UNKNOWN_CHART_TYPE}`. 메시지에 이미지 데이터를 넣지 않는다.

## 4. 동작

### 4.1 호스트 정상 흐름(S-01)

1. 사용자: "C:\samples\sfr_01.bmp 분석해줘".
2. 호스트 LLM → `analyze_image(path)`. 서버: 로드 → `filename_regex`로 `sfr_iso12233` → 기본 평가 → `proj_sfr` DLL 호출 → 판정 → `result.json`·`preview.jpg` 저장 → JSON 반환.
3. 호스트 LLM: `status`, `classification`, `judgement`를 읽고 요약·해석·권고 3절을 작성 → `write_report(image_id, …)`.
4. 서버: 표 + 서술을 합쳐 `report.md` 저장, 경로 반환.
5. 호스트 LLM: 경로·차트 유형·합격/불합격 건수·문제점을 사용자에게 답한다.

이미지당 도구 호출 2회가 기본이다. `unknown`이나 `needs_input`이면 `get_preview` 또는 사용자 확인 후 `analyze_image` 재호출로 1~2회 추가된다.

### 4.2 예외 흐름

| 상황 | 서버 동작 | 프롬프트가 지시하는 LLM 동작 |
|---|---|---|
| raw 메타 없음, 추정 신뢰도 낮음 | `needs_input: true`, 후보 반환, 상태 `partial` | 후보를 사용자에게 보여주고 확인 후 인자 지정 재호출 |
| 판별 `unknown` | `needs_classification: true`, 프리뷰 저장 | `get_preview`로 보고 카탈로그에서 선택, 확신 없으면 사용자에게 질문, `chart_type` 지정 재호출 |
| DLL 실패 | `dll_status: failed`, 원인, 기본 평가 유지 | 오류를 그대로 알리고 기본 평가로 보고서 작성 |
| 없는 `chart_type`·`analysis_id` | 오류 + 허용 목록 | 허용 목록에서 다시 선택 |
| 지원하지 않는 확장자 | 오류 | 사용자에게 알리고 다음 파일 |
| `write_report` 서술 누락 | 템플릿 문장 사용 | — |

### 4.3 호출 수와 지연

- thinking을 끌 수 없으므로 도구 호출 1회마다 수 초에서 수십 초가 든다. 이미지당 2~3회로 묶는 매크로 도구 설계가 지연 대책의 핵심이다(RISK-04).
- 도구 결과 JSON은 2~4KB로 제한한다. 히스토그램 같은 큰 배열은 파일에만 남긴다.
- 폴더 처리(P1)는 호스트 루프가 파일마다 2회씩 호출하므로 시연에서는 3~5장으로 제한한다.

## 5. GLM 연동 가이드: 사내 호스트 프로그램 경유

### 5.1 호스트 확인 사항

해커톤 시작 직후 30분 안에 확인한다(Q-07).

1. MCP 서버 등록 메뉴 또는 설정 파일이 있는가. 있으면 연동 사다리 A.
2. 없다면 로컬 명령 실행이나 파일 읽기(코워크형)가 되는가. 되면 B.
3. 둘 다 없으면 C(채팅 전용). 별도 API 안내가 있으면 D.
4. 도구 결과로 돌려준 이미지를 모델이 보는가(A-08). 안 보이면 프리뷰 판별은 사용자가 이미지를 채팅에 첨부하는 방식으로 바꾼다.
5. 시스템 프롬프트·프로젝트 지침을 넣을 수 있는 자리가 있는가. 없으면 대화 첫 메시지로 넣는다.
6. 사내 프로그램의 데이터 보존 정책이 과제 이미지의 보안 등급에 맞는가(NFR-02).

### 5.2 MCP 서버 등록

클로드 데스크톱 계열은 JSON 설정으로 stdio 서버를 등록한다. 사내 프로그램의 설정 파일 위치만 바꾼다.

```json
{
  "mcpServers": {
    "chart-agent": {
      "command": "C:\\work\\ai_agent\\.venv\\Scripts\\python.exe",
      "args": ["-m", "chart_agent.mcp_server"],
      "env": {
        "CHART_AGENT_REGISTRY": "C:\\work\\ai_agent\\config\\registry.yaml",
        "CHART_AGENT_OUT": "C:\\work\\ai_agent\\out"
      }
    }
  }
}
```

등록 후 호스트를 재시작하고 "chart-agent 도구 목록을 보여줘"로 5개 도구가 보이는지 확인한다.

### 5.3 호스트 시스템 프롬프트 전문

프로젝트 지침 또는 시스템 프롬프트 자리에 그대로 넣는다. 카탈로그 부분은 `get_chart_catalog` 결과가 우선한다.

```text
# 역할
당신은 카메라 화질 평가용 차트 이미지 분석 에이전트다. chart-agent 도구 서버의 도구만 사용해 분석하고,
수치는 절대 직접 계산하거나 추정하지 않는다.

# 절차 (이미지 1장마다 이 순서를 지킨다)
1. 경로가 폴더이면 list_inputs(path)로 파일 목록을 얻고 파일마다 2~5를 반복한다.
2. analyze_image(path)를 호출한다. 사용자가 차트 유형·과제·raw 사양을 말했으면
   chart_type, project_id, width, height, bit_depth, bayer 인자를 함께 넘긴다.
3. 결과를 확인하고 필요한 경우에만 다음을 수행한다.
   - input.raw_meta.needs_input 이 true: candidates 를 사용자에게 보여주고 폭·높이·비트 깊이·
     베이어 패턴을 확인한 뒤 그 값을 인자로 analyze_image 를 다시 호출한다.
   - classification.chart_type 이 "unknown": get_preview(image_id)로 프리뷰를 본 뒤
     카탈로그에서 가장 맞는 유형을 고른다. 확신이 없으면 후보 2개를 사용자에게 제시하고 묻는다.
     유형이 정해지면 chart_type 을 지정해 analyze_image 를 다시 호출한다.
   - project.dll_status 가 "failed": error 를 그대로 사용자에게 알리고 기본 평가 결과로 계속한다.
4. judgement 와 basic 을 근거로 한국어 서술 3개를 작성한다.
   - summary: 3문장 이내. 차트 유형, 전체 상태, 불합격 건수.
   - interpretation: passed 가 false 또는 null 인 항목 중심. 각 항목의 value 와 rule 을 그대로 인용.
   - recommendation: 재촬영·초점·노출·조명 조정 등 다음 행동 제안. 근거가 된 항목을 명시.
   작성 후 write_report(image_id, summary, interpretation, recommendation)를 호출한다.
5. 사용자에게 report_path, 차트 유형, 합격/불합격 건수, 발견한 문제를 5줄 이내로 답한다.

# 차트 카탈로그 (get_chart_catalog 결과가 우선한다)
- sfr_iso12233: ISO 12233 해상도(SFR) 차트. 흰 배경에 검은 기울어진 사각형(slanted edge) 여러 개. 과제 proj_sfr
- flat_field: 균일한 회색·흰색 면. 과제 없음
- unknown: 위에 해당하지 않음

# 규칙
- 모든 수치는 도구 결과에서만 인용한다. 단위와 소수점을 그대로 옮기고 새 수치를 만들거나 환산하지 않는다.
- 도구 오류는 숨기지 않고 사용자에게 알린다.
- 도구는 한 번에 하나만 호출하고 결과를 확인한 뒤 다음을 호출한다.
- 사용자가 chart_type 또는 project_id 를 지정하면 판별 단계를 생략한다.
- 도구 결과에 없는 분석을 요구받으면 "등록되지 않은 분석"이라고 답하고 get_chart_catalog 의 목록을 보여준다.
- 응답과 보고서는 한국어로 쓴다.
```

### 5.4 도구 설명문

FastMCP 도구 docstring에 그대로 넣는다. 설명문은 모델이 읽는 프롬프트다.

| 도구 | 설명문 |
|---|---|
| `list_inputs` | 폴더 또는 파일 경로를 받아 분석 가능한 파일(.bmp .jpg .jpeg .raw) 목록을 반환한다. 각 항목은 path, ext, size_bytes, has_sidecar다. 폴더를 분석할 때 가장 먼저 호출한다. |
| `get_chart_catalog` | 레지스트리에 등록된 차트 유형(id, name, description), 과제(id, name, chart_types), 분석 항목과 임계값을 반환한다. 차트 유형이 unknown일 때 유형을 고르는 기준이며, 사용자가 등록된 분석을 물을 때도 쓴다. |
| `analyze_image` | 이미지 1장을 로드 → 차트 판별(규칙) → 전체 이미지 기본 평가 → 과제 DLL 계산 → 임계값 판정까지 한 번에 실행하고 result JSON을 반환한다. raw는 width/height/bit_depth/bayer를 주지 않으면 사이드카 → 레지스트리 기본값 → 자동 추정 순으로 결정한다. 반환된 image_id는 get_preview와 write_report에 쓴다. classification.chart_type이 "unknown"이면 get_preview로 확인하고 chart_type을 지정해 다시 호출하라. input.raw_meta.needs_input이 true면 candidates를 사용자에게 확인한 뒤 인자를 지정해 다시 호출하라. 수치는 이 결과의 값만 인용하라. |
| `get_preview` | analyze_image가 만든 1024px 프리뷰 이미지를 반환한다. 차트 유형을 눈으로 판별하거나 사용자에게 보여줄 때 쓴다. |
| `write_report` | image_id의 수치 표·판정 표에 전달받은 summary, interpretation, recommendation을 합쳐 report.md를 저장하고 경로를 반환한다. 세 문장 묶음은 한국어로 쓰고 judgement 표의 수치만 인용한다. 분석 1장마다 마지막에 호출한다. |
| `run_analysis` (P2) | 이미 분석된 image_id에 대해 레지스트리에 등록된 분석 항목 하나를 지정 파라미터로 다시 실행하고 그 결과만 반환한다. 사용자가 특정 ROI나 항목을 추가로 물을 때 쓴다. analysis_id는 get_chart_catalog 목록에 있어야 한다. |

### 5.5 보고서 서술 지시의 근거

- 3절(요약·해석·권고)로 고정하면 `write_report` 인자와 1:1로 대응해 파싱이 필요 없다.
- "value와 rule을 그대로 인용"은 AC-07의 보조 장치다. 수치 표 자체는 코드가 넣으므로 LLM 문장이 틀려도 정본은 보존된다.
- "5줄 이내 답변"은 thinking 모델의 장황한 출력을 막아 시연 시간을 지킨다.

### 5.6 판별 보조 지시의 근거

- 코드가 VLM을 호출하지 않고 프롬프트의 절차 3으로 호스트 LLM이 프리뷰를 보게 한다. 구조화 출력·스키마 검증 코드가 필요 없다.
- "확신이 없으면 후보 2개를 제시하고 묻는다"가 오판별보다 미판별을 유도한다(AC-03 오판별 0건).
- 선택 결과는 `chart_type` 인자로 들어오므로 서버가 레지스트리로 검증한다.

### 5.7 채팅 전용 대체 경로(연동 사다리 C)

호스트가 도구를 못 쓰면 CLI가 `out/<image_id>/interpret_prompt.md`를 만든다. 내용은 5.3의 "절차 4"와 "규칙" + 수치 표 + 판정 표 + "아래 형식으로 답하라: `## 요약` `## 해석` `## 권고`"다. 사용자가 채팅에 붙여넣고 답을 `narrative.md`로 저장한 뒤 `chart-agent attach-narrative <image_id> narrative.md`를 실행하면 `report.md`가 완성된다.

### 5.8 GLM-5.3-Flash 특성에 맞춘 주의점

| 특성 | 대응 |
|---|---|
| thinking 항상 켜짐, 호출당 지연 | 매크로 도구로 호출 수 최소화, 도구 결과 2~4KB, 답변 길이 제한 |
| 병렬 도구 호출 미지원 가능 | 프롬프트에 "한 번에 하나만 호출" 명시, 서버는 호출 순서를 가정하지 않음 |
| 함수 호출 인자 자유도 | `chart_type`·`project_id`·`bayer`는 서버가 레지스트리 값으로 검증하고 오류에 허용 목록을 넣어 자기 수정 유도 |
| 수치 환각 | 수치 표는 코드 삽입, 프롬프트에 인용 규칙, 보고서 머리에 정본 표기 |
| 긴 컨텍스트(1M) | 카탈로그·결과를 넉넉히 넣어도 되지만 지연 때문에 줄인다 |
| 한국어 | 프롬프트·설명문·보고서 모두 한국어로 통일 |

### 5.9 시연 대사

1. "chart-agent 도구 목록 보여줘" → 5개 도구 확인.
2. "C:\demo\sfr_01.bmp 분석해줘" → 2회 호출 후 보고서 경로·판정 요약.
3. "C:\demo\unknown_01.jpg 분석해줘" → unknown → 프리뷰 확인 → 유형 질문 → 지정 후 재분석(P1).
4. "C:\demo\sensor_01.raw 분석해줘" → 추정 후보 확인 → 사양 지정 → 분석(P1).
5. "report.md 열어서 불합격 항목 설명해줘" → 해석.
6. 레지스트리에 `flat_field` 과제 임계값을 추가하고 "다시 분석해줘" → 설정만으로 동작(P1, AC-08).

## 6. 품질 속성

| 속성 | 설계 |
|---|---|
| 보안(NFR-02) | 이미지·결과는 로컬 `out/`에만. 호스트에는 1024px 프리뷰만. 키·비밀 없음. 호스트 데이터 정책은 5.1-6에서 확인 |
| 재현성(NFR-03) | 코어는 LLM 무관. `result.json`에 레지스트리 파일 해시 기록 |
| 안정성(NFR-04) | DLL 예외·반환 코드 처리(P0). 크래시 격리 워커는 P2. 리허설에서 크래시가 나면 앞당김 |
| 성능(NFR-05) | 이미지당 도구 호출 2~3회, 결과 JSON 소형화. 리허설에서 1장 2분 확인 |
| 관측성(NFR-06) | `run.log`(JSON Lines): 도구명·인자·판별·DLL 상태·소요 시간(P1) |
| 유지보수성 | 레지스트리 주도, 어댑터 1파일 = 과제 1개, 프롬프트는 `prompts/` 텍스트 파일 |
| 테스트 가능성(NFR-07) | mock DLL, 합성 차트 생성기(흰 배경 + 기울어진 검은 사각형), CLI 단독 실행 |

## 7. 검증 전략

| 인수 조건 | 전략(방법 선택은 wf-implement) |
|---|---|
| AC-01, AC-02 | 합성 bmp/jpg/raw 픽스처 단위 테스트. raw는 크기 불일치 케이스 포함 |
| AC-03 | 샘플 전부를 CLI로 돌려 판별 표 작성(규칙). 호스트 판별은 리허설 기록 |
| AC-04, AC-05 | mock 정상·예외 두 가지로 파이프라인 테스트 |
| AC-06 | `chart-agent analyze` 단독 실행 확인 |
| AC-07 | 보고서 표 파싱 vs `result.json` 비교 테스트 |
| AC-08 | 테스트 레지스트리에 유형·과제 추가 후 코어 변경 없이 통과 |
| AC-09 | 없는 ID로 도구 호출 시 오류·허용 목록 확인 |
| AC-10 | 2회 실행 수치 비교 |
| AC-11 | 호스트 리허설: 1회 지시로 보고서 생성, 화면 녹화 |
| AC-12 | 사양을 아는 raw로 추정 결과 비교 |

## 8. 기술 스택과 디렉터리 구조

- Python 3.11, `numpy`, `pillow`, `pydantic`, `pyyaml`, `opencv-python`, `mcp`, `pytest`. 외부 Agent 프레임워크 없음. 해커톤 장소에서 pip가 안 되면 휠을 미리 받아 둔다(A-09).

```text
ai_agent/
├── docs/
├── config/registry.yaml
├── prompts/{system_prompt.md, interpret_prompt_template.md}
├── src/chart_agent/
│   ├── cli.py              # DES-01
│   ├── pipeline.py         # DES-02
│   ├── models.py           # 3.1
│   ├── registry.py         # DES-05
│   ├── loaders/{image.py, raw.py, raw_estimate.py}   # DES-03
│   ├── classify.py         # DES-04
│   ├── basic.py            # DES-06
│   ├── dll/{base.py, adapters/sfr.py, adapters/mock_sfr.py}   # DES-07
│   ├── judge.py, report.py # DES-09
│   └── mcp_server.py       # DES-08
├── tests/{test_loaders.py, test_basic.py, test_pipeline.py, test_report.py, fixtures/}
├── demo/                   # 시연 이미지
└── pyproject.toml
```

## 9. 12시간 해커톤 진행 순서

### 9.1 사전 준비(해커톤 전, 규칙이 허용하는 범위에서)

- 이 문서와 [요구사항](./requirements.md)을 반입한다. 빌드 프롬프트는 이 문서 절 번호를 참조한다.
- 샘플 확보: SFR 차트 bmp/jpg 2장 이상, raw 1장 이상(사양 메모), 차트가 아닌 이미지 1장.
- DLL 헤더 또는 함수 시그니처 확보(Q-08). 못 구하면 mock으로 진행한다.
- 휠 파일 다운로드: `numpy pillow pydantic pyyaml opencv-python mcp pytest`.
- 사내 프로그램의 MCP 설정 파일 위치 확인(Q-07).

### 9.2 타임라인

| 시간 | 단계 | 산출물 | 빌드 프롬프트 |
|---|---|---|---|
| 0:00–0:30 | 환경 확인: venv, 설치, DLL 로드 테스트, 호스트 확장 방식 확인(5.1) | 확인 메모 | B0 |
| 0:30–2:30 | 코어 1: 모델·레지스트리·로더(bmp/jpg/raw 지정 메타)·기본 평가·CLI | `result.json` 생성 | B1, B2 |
| 2:30–4:00 | 코어 2: DLL ctypes 어댑터 + mock, 판정, 템플릿 보고서 | `report.md` 생성(LLM 없음) | B3 |
| 4:00–5:30 | MCP 서버 5개 도구, 호스트 등록, `analyze_image` 1회 성공 | 도구 목록 표시 | B4 |
| 5:30–6:30 | 시스템 프롬프트 투입, E2E 1장 시연(AC-11) | **P0 완료 체크포인트** | — |
| 6:30–8:00 | P1: raw 추정, 파일명 규칙 판별, unknown → 프리뷰 → 호스트 판별 | S-03, S-04 | B5 |
| 8:00–9:30 | P1: 폴더 일괄, 오류 경로(DLL 실패·크기 불일치), 보고서 문구, run.log | S-02, 예외 표 | B6 |
| 9:30–10:30 | 안정화·README·시연 데이터 정리 | README | B7 |
| 10:30–11:30 | 리허설 2회(5.9 대사), 지연 측정, 프롬프트 다듬기 | 리허설 기록 | — |
| 11:30–12:00 | 발표 자료(구조도 2.1, 시연 녹화) | 발표 | — |

### 9.3 빌드 프롬프트(코딩 보조 AI에게 순서대로)

아래는 요지다. 사내 LLM이 이 설계 문서를 못 읽는 상황을 위해 사양을 안에 담은 자기완결 전문은 [해커톤 프롬프트 팩](./hackathon-prompt-pack.md) 01~08에 있다.

- **B0** — "Python 3.11 가상환경을 만들고 numpy pillow pydantic pyyaml opencv-python mcp pytest를 설치하는 스크립트를 작성해라. 그리고 `ctypes.WinDLL(r'<DLL 경로>')`로 로드해 함수 `<이름>`이 존재하는지, 64비트인지 출력하는 `check_dll.py`를 작성해 실행해라."
- **B1** — "설계 문서 3.1의 pydantic 모델과 3.2의 레지스트리 YAML 로더(`registry.py`)를 구현해라. 로더는 `loaders/image.py`(bmp/jpg → 8비트 RGB numpy)와 `loaders/raw.py`(헤더 없는 베이어 덤프, width/height/bit_depth/bayer/packing=unpacked16, 파일 크기 검증, OpenCV 디모자이크, 1024px JPEG 프리뷰)로 나눈다. 합성 이미지를 만드는 pytest 픽스처와 테스트를 포함해라. 설계 3.7의 오류 코드를 쓴다."
- **B2** — "설계 DES-06 표의 7개 기본 평가 항목을 `basic.py`에 결정적으로 구현하고, `pipeline.run(path, chart_type=None, project_id=None, raw_override=None) -> AnalysisResult`와 `cli.py analyze` 명령을 만들어 `out/<image_id>/result.json`과 `preview.jpg`를 쓰게 해라. 판별은 사용자 지정 → 레지스트리 filename_regex → unknown 순이다(DES-04). LLM 호출은 없다."
- **B3** — "설계 3.5의 `DllAdapter` 프로토콜, `adapters/sfr.py`(ctypes, 시그니처는 다음과 같다: …), `adapters/mock_sfr.py`(같은 call 시그니처, 중앙 ROI 라플라시안 분산 기반 결정적 pseudo-MTF)를 구현해라. 레지스트리 `dll.kind`가 `c`면 실제, `mock`이면 mock을 쓴다. DLL 예외·반환 코드 ≠ 0은 `dll_status: failed`로 기록하고 파이프라인은 계속된다. `judge.py`는 3.2의 thresholds 문법(`>= 0.3`, `between a b`)을 평가한다. `report.py`는 DES-09대로 표 + 서술 3절(없으면 템플릿)을 `report.md`로 쓴다."
- **B4** — "`mcp_server.py`를 FastMCP로 작성해라. 도구는 설계 3.3의 `get_chart_catalog`, `analyze_image`, `write_report`, `get_preview`, `list_inputs`이고 docstring은 5.4의 설명문을 그대로 쓴다. `chart_type`·`project_id`·`bayer`는 레지스트리 값으로 검증하고 아니면 허용 목록을 담은 오류를 반환한다. 반환 JSON은 히스토그램을 제외해 4KB 이하로 유지한다. 5.2 형식의 호스트 설정 JSON도 생성해라."
- **B5** — "설계 3.6의 raw 메타 추정을 `loaders/raw_estimate.py`에 구현해라. `RawMeta.estimated/confidence/candidates/needs_input`을 채우고, 사양을 아는 raw 픽스처로 폭·높이·비트가 맞는지 테스트해라. `analyze_image`가 raw 인자 없이 호출되면 사이드카 → 레지스트리 기본값 → 추정 순으로 쓰게 연결해라."
- **B6** — "폴더 입력(`list_inputs`, CLI 폴더 처리, `out/summary.json`), 오류 경로 테스트(DLL 예외 mock, 크기 불일치 raw, 없는 analysis_id), `run.log` JSON Lines 기록을 추가해라. `report.md` 템플릿 문장을 다듬고, 보고서 표의 수치가 `result.json`과 일치하는지 검사하는 테스트를 추가해라."
- **B7** — "README에 설치, 호스트 등록(5.2), 시스템 프롬프트 투입(5.3), 시연 대사(5.9), 레지스트리에 과제 추가하는 방법을 써라. `demo/` 폴더 구성과 리허설 체크리스트를 만들어라."

### 9.4 체크포인트 규칙

- 6:30에 AC-11(호스트에서 1회 지시로 보고서 생성)이 안 되면 P1을 모두 중단하고 P0 안정화에 쓴다.
- 실제 DLL이 로드되지 않으면 mock으로 시연하고 발표에서 "어댑터 1파일 교체"임을 설명한다(RISK-08).
- 호스트가 MCP를 지원하지 않으면 4:00 시점에 연동 사다리 B 또는 C로 전환한다. 코어·CLI는 그대로 쓴다.

## 추적성

| 요구사항 | 설계 요소 | 인수 조건 | 검증 |
|---|---|---|---|
| [FR-01](./requirements.md#4-기능-요구사항) | DES-03 | AC-01, AC-02 | TBD — wf-implement에서 VER 발행 |
| [FR-02](./requirements.md#4-기능-요구사항) | DES-04, DES-10 절차 3, 5.6 | AC-03 | TBD — 동상 |
| [FR-03](./requirements.md#4-기능-요구사항) | DES-05, DES-08 검증 | AC-09 | TBD — 동상 |
| [FR-04](./requirements.md#4-기능-요구사항) | DES-06 | AC-01, AC-10 | TBD — 동상 |
| [FR-05](./requirements.md#4-기능-요구사항) | DES-07, 3.5 | AC-04, AC-05 | TBD — 동상 |
| [FR-06](./requirements.md#4-기능-요구사항) | DES-09, 5.5 | AC-06, AC-07 | TBD — 동상 |
| [FR-07](./requirements.md#4-기능-요구사항) | DES-01, DES-08, 2.3 | AC-06, AC-11 | TBD — 동상 |
| [FR-08](./requirements.md#4-기능-요구사항) | DES-05, 3.2 | AC-08 | TBD — 동상 |
| [FR-09](./requirements.md#4-기능-요구사항) | DES-10 절차 3 | AC-11 | TBD — 리허설 |
| [FR-10](./requirements.md#4-기능-요구사항) | DES-03, 3.6 | AC-12 | TBD — 동상 |
| [FR-11](./requirements.md#4-기능-요구사항) | DES-02, `list_inputs` | — | TBD — 리허설 |
| [FR-12](./requirements.md#4-기능-요구사항) | `run_analysis` | — | N/A — P2 |
| [NFR-01](./requirements.md#5-비기능-요구사항) | 1절 원칙 1, 2.3, 2.4 | AC-06, AC-11 | TBD — 동상 |
| [NFR-02](./requirements.md#5-비기능-요구사항) | 6절 보안, 5.1-6 | — | TBD — 코드 리뷰 |
| [NFR-03](./requirements.md#5-비기능-요구사항) | ADR-001, DES-09 | AC-07, AC-10 | TBD — 동상 |
| [NFR-04](./requirements.md#5-비기능-요구사항) | DES-07, ADR-002 | AC-05 | TBD — 동상 |
| [NFR-05](./requirements.md#5-비기능-요구사항) | 4.3, 9.2 리허설 | — | TBD — 리허설 |
| [NFR-06](./requirements.md#5-비기능-요구사항) | 6절 관측성 | — | TBD — 로그 검사 |
| [NFR-07](./requirements.md#5-비기능-요구사항) | DES-07 mock, 7절 | AC-04, AC-05 | TBD — 동상 |

## 승인 기록

| 일자 | 승인자 | 대상 | 결과 | 기준선 | 근거 |
|---|---|---|---|---|---|
| 2026-10-11 | 사용자 | 1차 초안 | 수정 요청 | — | Q-01~Q-06 답변과 해커톤 제약 |
| 2026-10-11 | 사용자 | 2차 초안(이 문서), [요구사항](./requirements.md), ADR-001~003 | `awaiting-approval` | — | 대화에서 재승인 요청, 응답 대기 |

## 인계

- 다음 단계 또는 워크플로우: 승인 후 [wf-implement](file:///C:/Users/JS/.claude/skills/wf-implement/SKILL.md) 계획 수립. 9.2 타임라인을 TASK로 분해한다.
- 시작 조건: 기준선 `v1` 발행, ADR-001~003 `approved`, Q-07 답변(없으면 A-06 가정으로 시작하고 5.1에서 확인)
- 입력 문서와 기준선: [REQ-image-analysis-agent](./requirements.md), 이 문서, [ADR-001](./work/20261011-image-analysis-agent/ADR-001-llm-role-split.md), [ADR-002](./work/20261011-image-analysis-agent/ADR-002-dll-worker-process.md), [ADR-003](./work/20261011-image-analysis-agent/ADR-003-llm-host-mcp-integration.md)
- 완료된 항목: 요구사항 2차 초안, 설계 2차 초안, ADR 3건 개정, 시스템 프롬프트·도구 설명문·빌드 프롬프트 초안
- 미완료 항목: 사용자 승인, Q-07~Q-10 답변, 기준선 발행
- 차단 요인: 승인 전 구현 시작 불가
- 다음 행동: 사용자 응답 반영 후 상태 갱신

## 변경 이력

| 일자 | 변경 | 근거 | 상태 |
|---|---|---|---|
| 2026-10-11 | 최초 작성(API 클라이언트 기반 Agent) | 요구사항 1차 초안 | `awaiting-approval` |
| 2026-10-11 | 2차 초안: 실행 주체를 사내 호스트 프로그램으로 변경, MCP 도구 서버·연동 사다리·매크로 도구 도입, LLM 클라이언트 설계 제거, raw 추정 알고리즘·시스템 프롬프트 전문·도구 설명문·12시간 타임라인·빌드 프롬프트 추가 | 사용자 1차 답변, [요구사항](./requirements.md) 2차 초안 | `awaiting-approval` |
| 2026-10-12 | 부속 문서 [해커톤 프롬프트 팩](./hackathon-prompt-pack.md) 추가(자기완결 빌드 프롬프트 00~08, 런타임 20~24, 운영 30~33)와 링크 | 사용자 요청(해커톤에서 클로드 사용 불가) | `awaiting-approval` |
