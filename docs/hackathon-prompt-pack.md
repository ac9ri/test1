# DESIGN-image-analysis-agent-prompts: 해커톤 프롬프트 팩

> 문서 유형: `design`
> 작업 ID: `20261011-image-analysis-agent`
> 상태: `superseded`
> 기준선: `N/A — 대체된 초안`
> 작성일: 2026-10-11
> 최종 갱신: 2026-10-13
> 관련 문서: [DESIGN-image-analysis-agent-prompts-web: 웹 해커톤 프롬프트 팩](./hackathon-prompt-pack-web.md)(대체 문서), [DESIGN-image-analysis-agent: 설계](./design.md)

## 요약

- **이 문서는 2026-10-13에 대체되었다.** 개발 목표가 MCP 도구 서버에서 웹 개발로 바뀌어 [웹 해커톤 프롬프트 팩](./hackathon-prompt-pack-web.md)이 현행 문서다. 아래 내용은 MCP 호스트 연동 설계의 이력으로만 보존한다.
- 목적(당시): 해커톤에서 클로드 없이 사내 LLM(GLM-5.3-Flash, 사내 호스트 프로그램)만으로 Agent를 만들고 운용하기 위해, 붙여넣을 프롬프트 전부를 미리 확정한다.
- 현재 결론 또는 상태: 프롬프트 3묶음. **빌드용 00~08**은 사내 LLM에게 코드를 쓰게 하는 프롬프트, **런타임용 20~24**는 호스트에 넣는 시스템 프롬프트·도구 설명문·대체 경로, **운영용 30~33**은 시연 대사·문제 해결·자체 점검·컨텍스트 복구다. [설계](./design.md) 5절과 9.3절의 전문 확장판이다.
- 다음 행동: 해커톤 전에 [자리표시자](#자리표시자)를 채운다. 당일에는 `00 브리프`를 새 대화마다 먼저 붙여넣고 `01`부터 순서대로 쓴다.

## 문서 연결

| 방향 | 관계 | 대상 문서 | 대상 항목 | 비고 |
|---|---|---|---|---|
| input | baseline | [DESIGN-image-analysis-agent: 설계](./design.md) | DES-10, 3.1~3.7, 5.3, 5.4, 9.3 | 이 문서는 설계의 프롬프트 부속이다 |
| input | baseline | [REQ-image-analysis-agent: 요구사항](./requirements.md) | FR-01~FR-11, AC-01~AC-12 | 자체 점검 프롬프트의 기준 |

## 사용 방법

1. **해커톤 전:** 아래 자리표시자 표를 채우고 이 문서를 반입한다. 휠 파일을 `wheels/`에 받아 둔다.
2. **당일 새 대화마다:** `00 프로젝트 브리프`를 첫 메시지로 붙여넣는다. 대화가 길어져 느려지면 새 대화를 열고 `33 컨텍스트 복구`로 이어 간다.
3. **빌드:** `01`→`08` 순서. 각 프롬프트 끝의 완료 조건을 확인하고 넘어간다. 4시간 시점 체크포인트에서 MCP 등록이 안 되면 `23`·`24` 대체 경로로 간다.
4. **런타임:** 호스트에 `22` 설정 JSON으로 서버를 등록하고 `20` 시스템 프롬프트를 프로젝트 지침 자리에 넣는다. `21` 도구 설명문은 이미 코드 docstring에 들어가 있다.
5. **시연:** `30` 대사 순서대로. 막히면 `31`, 코드 점검은 `32`.

## 자리표시자

| 자리표시자 | 의미 | 예 |
|---|---|---|
| `<<WORK_DIR>>` | 작업 폴더 | `C:\hack\ai_agent` |
| `<<PYTHON_EXE>>` | 가상환경 파이썬 | `C:\hack\ai_agent\.venv\Scripts\python.exe` |
| `<<DLL_PATH>>` | SFR DLL 경로 | `C:\tools\sfr\SfrCalc.dll` |
| `<<DLL_FUNCTION>>` | 호출할 C 함수명 | `CalcSFR` |
| `<<DLL_SIGNATURE>>` | 헤더의 C 시그니처 한 줄 | `int CalcSFR(const unsigned short* img, int w, int h, int x, int y, int rw, int rh, double* out2);` |
| `<<DLL_OUTPUTS>>` | 반환값 이름과 순서 | `mtf50, mtf30` |
| `<<DLL_INPUT_KIND>>` | `buffer`(픽셀 포인터) 또는 `path`(파일 경로 문자열) | `buffer` |
| `<<RAW_W>> <<RAW_H>> <<RAW_BITS>> <<RAW_BAYER>>` | 과제 raw 기본 사양 | `4000 3000 10 RGGB` |
| `<<SAMPLE_DIR>>` | 샘플 이미지 폴더 | `C:\hack\samples` |
| `<<HOST_MCP_CONFIG_PATH>>` | 호스트 프로그램의 MCP 설정 파일 | `%APPDATA%\<프로그램>\mcp.json` |

DLL 정보를 해커톤 전에 못 구하면 `<<DLL_INPUT_KIND>>`는 `buffer`, 시그니처는 예시 값으로 두고 mock으로 진행한다. 어댑터 1파일만 나중에 바꾼다.

---

## 00 프로젝트 브리프 (매 새 대화의 첫 메시지)

```text
[프로젝트 브리프 — 이 대화 내내 아래를 기준으로 답하라]

당신은 Python 시니어 엔지니어다. 나는 12시간 해커톤에서 "카메라 화질 평가용 차트 이미지 분석 AI Agent"를 만든다.
당신의 역할은 내가 주는 작업 프롬프트마다 완전한 코드를 쓰는 것이다. 설명은 짧게, 코드는 전체를 출력한다.

1. 목표
- 사내 호스트 프로그램(클로드 데스크톱 유사, MCP 서버 등록 가능)에서 사용자가 "C:\...\sfr_01.bmp 분석해줘"라고 하면
  호스트의 LLM이 우리 MCP 도구를 호출해 차트 판별 → 전체 이미지 기본 평가 → 과제 DLL(SFR 해상도) 계산 → 임계값 판정 →
  보고서 report.md 생성까지 끝낸다.
- 같은 코어를 CLI로도 실행한다. LLM 없이 result.json과 템플릿 보고서가 나와야 한다.
- 우선순위: P0 = 시연 필수, P1 = 시간이 남으면. 내가 P0라고 하면 P1 기능을 넣지 않는다.

2. 환경과 제약
- Windows 11, Python 3.11. 허용 패키지: numpy, pillow, pydantic(v2), pyyaml, opencv-python, mcp, pytest. 다른 패키지·프레임워크 금지.
- 과제 DLL은 C 함수이며 ctypes로 호출한다. 실제 DLL이 없으면 같은 호출 시그니처의 mock(Python)을 쓴다.
- raw는 헤더 없는 베이어 덤프. 16비트 little-endian unpacked(픽셀당 2바이트), 유효 비트 8/10/12/14/16, 패턴 RGGB/BGGR/GRBG/GBRG.
- 수치 계산은 전부 결정적 코드다. LLM은 판별 보조와 해석만 한다. 코드 안에서 LLM을 호출하지 않는다.

3. 디렉터리 (src 레이아웃, 패키지명 chart_agent, pyproject.toml에 콘솔 스크립트 chart-agent = chart_agent.cli:main)
  <<WORK_DIR>>/
    config/registry.yaml
    src/chart_agent/__init__.py, models.py, registry.py, errors.py, pipeline.py, cli.py, basic.py, classify.py,
                     judge.py, report.py, mcp_server.py
    src/chart_agent/loaders/__init__.py, image.py, raw.py, raw_estimate.py
    src/chart_agent/dll/__init__.py, base.py, adapters/__init__.py, adapters/sfr.py, adapters/mock_sfr.py
    tests/ (pytest, 합성 이미지 픽스처), scripts/, demo/, out/

4. 데이터 모델 (src/chart_agent/models.py) — 아래 정의를 그대로 쓴다
  class RawMeta(BaseModel):
      width: int; height: int; bit_depth: int
      bayer: Literal["RGGB","BGGR","GRBG","GBRG"]
      packing: Literal["unpacked16","unpacked8"] = "unpacked16"
      estimated: bool = False
      confidence: float = 1.0
      candidates: list[dict] = []     # {"width","height","bit_depth","bayer","score"}
      needs_input: bool = False
  class RawOverride(BaseModel):
      width: int | None = None; height: int | None = None; bit_depth: int | None = None; bayer: str | None = None
  class ImageInfo(BaseModel):
      path: str; format: Literal["bmp","jpg","raw"]
      width: int; height: int; channels: int; bit_depth: int
      sha256: str; raw_meta: RawMeta | None = None
  class ChartClassification(BaseModel):
      chart_type: str                 # 레지스트리 id 또는 "unknown"
      confidence: float
      method: Literal["user","rule","none"]
      needs_classification: bool = False
  class ProjectResult(BaseModel):
      project_id: str | None
      dll_status: Literal["ok","failed","skipped"]
      results: dict[str, float | int | str] = {}
      error: str | None = None
      elapsed_ms: int | None = None
  class Judgement(BaseModel):
      metric: str; value: float | int | str; rule: str | None; passed: bool | None
  class AnalysisResult(BaseModel):
      schema_version: Literal["1"] = "1"
      image_id: str                   # 파일 바이트 sha256 앞 12자리
      input: ImageInfo
      classification: ChartClassification
      basic: dict[str, dict]          # 항목 id → 값 dict
      project: ProjectResult
      judgement: list[Judgement]
      status: Literal["ok","partial","failed"]
      errors: list[dict] = []         # {"stage","code","message"}
      preview_path: str
      report_path: str | None = None
  상태 규칙: 모든 단계 성공 ok. unknown 차트, DLL 실패, raw 추정 사용 중 하나라도 있으면 partial. 로드 실패 failed.

5. 레지스트리 (config/registry.yaml) — 아래 형식. registry.py는 이를 pydantic으로 로드·검증한다
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
      raw_defaults: {width: <<RAW_W>>, height: <<RAW_H>>, bit_depth: <<RAW_BITS>>, bayer: <<RAW_BAYER>>}
      dll:
        kind: mock                   # c | mock
        path: "<<DLL_PATH>>"
        adapter: "chart_agent.dll.adapters.sfr:SfrAdapter"
        mock_adapter: "chart_agent.dll.adapters.mock_sfr:MockSfrAdapter"
      analyses:
        - id: proj_sfr.center
          function: <<DLL_FUNCTION>>
          params: {roi: center}
          thresholds: {mtf50: ">= 0.30", mtf30: ">= 0.45"}
  basic_thresholds:
    basic.clipping.saturated_ratio: "<= 0.01"
    basic.sharpness.center: ">= 50"
  임계값 문법: ">= x", "> x", "<= x", "< x", "== x", "between a b". 판정 metric 이름은 "<analysis_id>.<key>" 또는 "basic.<항목>.<key>".

6. 오류 (src/chart_agent/errors.py)
  class ChartAgentError(Exception): code: str, message: str, stage: str
  코드: E_UNSUPPORTED_FORMAT, E_RAW_META_MISSING, E_RAW_SIZE_MISMATCH, E_RAW_ESTIMATE_AMBIGUOUS,
        E_DLL_LOAD, E_DLL_CALL, E_DLL_NOT_REGISTERED, E_UNKNOWN_ANALYSIS, E_UNKNOWN_CHART_TYPE, E_UNKNOWN_PROJECT
  stage: load, classify, basic, dll, judge, report

7. 출력 규칙 (반드시 지켜라)
- 생성·변경하는 파일마다 "### 파일: <상대경로>" 제목 뒤에 파일 전체 내용을 하나의 코드 블록으로 출력한다. 일부만 출력하지 않는다.
- 마지막에 실행 명령과 테스트 명령을 적는다.
- 설명은 10줄 이내. 가정이 필요하면 코드 상단 주석에 적고 진행한다. 질문으로 멈추지 않는다.
- 함수는 작게, 타입 힌트, docstring 한 줄. 전역 가변 상태 금지. 로그는 logging 모듈. 비밀 정보 없음.
- 같은 파일을 다시 요청받으면 이전 내용과 호환되게 전체를 다시 출력한다.

이해했으면 "브리프 확인"이라고만 답하라.
```

---

## 01 작업 B0: 환경 확인 (0:00~0:30)

```text
[작업 B0: 환경 확인] P0

다음 세 파일을 만들어라.

1. setup.ps1
- <<WORK_DIR>>에서 python -m venv .venv 를 만들고 .venv\Scripts\python.exe -m pip install 로
  numpy pillow pydantic pyyaml opencv-python "mcp[cli]" pytest 를 설치한다.
- 변수 $Offline 이 $true 이면 --no-index --find-links .\wheels 옵션을 붙인다.
- 마지막에 pip install -e . 로 패키지를 편집 모드로 설치한다(pyproject.toml은 B1에서 만들지만 명령은 넣어 둔다).

2. scripts/check_dll.py
- 인자: DLL 경로, 함수명.
- (a) ctypes.WinDLL 로드 성공 여부 (b) getattr 로 함수 존재 여부 (c) PE 헤더(IMAGE_FILE_HEADER.Machine)를 읽어 32/64비트
  (d) 로드 실패 시 WinError 코드 해석: 193 = 비트 불일치, 126 = 의존 DLL 누락, 그 외는 원문 출력.
- 결과를 표 형태로 stdout에 출력하고, 실패면 종료 코드 1.

3. scripts/check_samples.py
- 인자: 폴더. .bmp/.jpg/.jpeg/.raw 파일마다 파일 크기, 이미지면 해상도·모드, raw면 "픽셀당 2바이트 가정 픽셀 수"와
  그 수와 정확히 일치하는 (w,h) 후보를 다음 표에서 찾아 출력:
  640x480, 1280x720, 1280x960, 1920x1080, 2592x1944, 3264x2448, 4000x3000, 4032x3024, 4608x3456, 8000x6000.

완료 조건: 세 파일이 실행되고 사용법이 각 파일 docstring에 있다.
```

---

## 02 작업 B1: 모델·레지스트리·로더 (0:30~1:30)

```text
[작업 B1: 모델, 레지스트리, 로더] P0

브리프 4절의 모델과 5절의 레지스트리 형식을 그대로 구현하라. 파일: pyproject.toml, src/chart_agent/__init__.py,
models.py, errors.py, registry.py, loaders/__init__.py, loaders/image.py, loaders/raw.py, tests/conftest.py,
tests/test_loaders.py, config/registry.yaml(브리프 5절 내용).

registry.py
- Registry.load(path) -> Registry. 중복 id, 없는 project 참조, 임계값 문법 오류는 ValueError 로 메시지와 함께 실패.
- 조회: chart_type(id), project(id), project_for_chart(chart_id), analyses_for(project_id), all_analysis_ids(),
  catalog_dict() -> {"chart_types":[{id,name,description,project}], "projects":[{id,name,analyses:[{id,thresholds}]}], "basic_thresholds":{...}}.
- 환경 변수 CHART_AGENT_REGISTRY 가 있으면 기본 경로로 쓴다. out_dir 는 레지스트리 파일 기준 상대 경로를 절대 경로로 바꾼다.

loaders/image.py
- load_bmp_jpg(path) -> (rgb8: np.ndarray (H,W,3) uint8, info: ImageInfo). Pillow 로 열고 convert("RGB").
  원본 mode 가 L 이면 channels=1, 아니면 3. bit_depth=8. format 은 확장자로 bmp|jpg.

loaders/raw.py
- resolve_raw_meta(path, override: RawOverride|None, project_defaults: dict|None) -> RawMeta|None
  우선순위: override 의 값이 전부 있으면 그것 > 사이드카 "<path>.raw.json"(키 width,height,bit_depth,bayer) > project_defaults > None.
  override 가 일부만 있으면 나머지를 사이드카·기본값으로 채운다.
- load_raw(path, meta: RawMeta) -> (bayer_u16: np.ndarray (H,W), rgb8: np.ndarray (H,W,3), info: ImageInfo)
  기대 크기 = width*height*2 (unpacked16) 또는 width*height (unpacked8). 다르면 ChartAgentError("E_RAW_SIZE_MISMATCH",
  f"expected {expected} bytes, got {actual}", "load"). np.fromfile(dtype="<u2") 후 reshape.
  디모자이크는 OpenCV 를 쓰되 패턴 이름 매핑은 반드시 다음을 따른다(OpenCV 명명이 반대다):
    RGGB -> cv2.COLOR_BayerBG2RGB, BGGR -> cv2.COLOR_BayerRG2RGB, GRBG -> cv2.COLOR_BayerGB2RGB, GBRG -> cv2.COLOR_BayerGR2RGB
  uint16 입력 그대로 cvtColor 하고, 결과를 (2**bit_depth - 1) 로 나눠 0..255 로 정규화해 uint8 로 만든다.
- make_preview(rgb8, out_path, long_edge=1024, stretch=False): 긴 변 1024 로 LANCZOS 리사이즈 후 JPEG 품질 85 저장.
  stretch=True 면 1~99 퍼센타일로 대비를 늘린다(raw 프리뷰 전용. 수치 계산에는 절대 쓰지 않는다).

loaders/__init__.py
- @dataclass LoadedImage: native: np.ndarray (raw 면 bayer u16, 아니면 rgb8), rgb8: np.ndarray, info: ImageInfo
- load_any(path, override, project_defaults) -> LoadedImage. 확장자 외는 E_UNSUPPORTED_FORMAT. raw 인데 meta 가 None 이면
  E_RAW_META_MISSING(메시지에 "width,height,bit_depth,bayer 가 필요하다" 포함). sha256 는 파일 바이트 전체.

tests
- conftest: 64x48 RGB 합성 이미지(왼쪽 반 흰색, 오른쪽 반 검은색)를 bmp/jpg 로 저장하는 픽스처,
  32x24 RGGB 베이어 u16(10비트, R=800 G=500 B=200 상수)을 .raw 로 저장하는 픽스처.
- test: bmp/jpg 로드 크기·채널, raw 로드 후 rgb8 의 R>G>B 순서 확인, 사이드카 우선순위, 크기 불일치 예외 코드.

완료 조건: pytest tests/test_loaders.py 통과. 실행 명령: .venv\Scripts\python.exe -m pytest -q
```

---

## 03 작업 B2: 기본 평가·파이프라인·CLI (1:30~2:30)

```text
[작업 B2: 기본 평가, 파이프라인, CLI] P0

파일: src/chart_agent/basic.py, classify.py, pipeline.py, cli.py, tests/test_basic.py, tests/test_pipeline.py.
B1 의 모델·로더를 그대로 쓴다.

basic.py — run_basic(img: LoadedImage) -> dict[str, dict]. 전부 numpy/OpenCV, 결정적. 소수는 round(x, 4).
  gray = cv2.cvtColor(rgb8, COLOR_RGB2GRAY)
  "basic.info":       {width, height, channels, bit_depth, format}
  "basic.stats":      {r_mean, r_std, g_mean, g_std, b_mean, b_std, min, max, luma_mean}  (rgb8 기준)
  "basic.clipping":   {saturated_ratio: gray==255 비율, dark_ratio: gray==0 비율}
  "basic.sharpness":  {center: 중앙 ROI(가로세로 각 50%, 중앙 정렬)의 라플라시안 분산, full: 전체}  cv2.Laplacian(gray, cv2.CV_64F).var()
  "basic.noise":      {sigma: 1.4826 * median(|gray - medianBlur(gray,3)|)}
  "basic.uniformity": {grid: 3x3 각 블록 평균 9개 리스트(행 우선), corner_center_ratio: 네 모서리 평균의 최소 / 중앙 평균}
  raw 이면 추가 "basic.bayer_stats": {r_mean, gr_mean, gb_mean, b_mean, r_std, gr_std, gb_std, b_std} (native u16, 패턴에 따라 위상 매핑)

classify.py — classify(path, user_chart_type, registry) -> ChartClassification
  순서: user_chart_type 지정(레지스트리에 있어야 함, 없으면 E_UNKNOWN_CHART_TYPE) → method "user", 1.0
        파일명(stem)에 chart_types[*].filename_regex 가 search 되면 → method "rule", 1.0 (여러 개면 첫 번째)
        아니면 "unknown", 0.0, method "none", needs_classification True

pipeline.py — run(path, chart_type=None, project_id=None, raw_override=None, registry=None, out_dir=None) -> AnalysisResult
  1) registry 기본 로드. 2) project_id 가 없고 chart_type 이 정해지면 레지스트리의 project 를 쓴다(판별 전에는 모르므로
     raw 기본값은 project_id 인자 또는 파일명 규칙으로 먼저 판별한 뒤 결정한다. 순서: classify → project → load).
  3) load_any. 4) run_basic. 5) project 가 있으면 B3 의 run_project 를 호출(이 작업에서는 ProjectResult(dll_status="skipped") 스텁).
  6) 판정은 B3 에서. 여기서는 judgement=[]. 7) out_dir/<image_id>/ 에 preview.jpg 와 result.json 저장. 8) 상태 규칙 적용.
  단계별 예외는 ChartAgentError 면 errors 에 추가하고 가능한 한 계속한다. load 실패만 failed 로 즉시 종료.

cli.py — argparse. 명령: analyze <path> [--chart-type] [--project] [--width --height --bit-depth --bayer] [--out] [--registry]
  validate-registry [--registry]. 종료 코드 0 성공·부분 성공, 1 예기치 않은 오류, 2 입력·설정 오류.
  analyze 는 result.json 경로와 status 를 출력한다. main() 을 콘솔 스크립트 진입점으로.

tests
- test_basic: 왼쪽 흰/오른쪽 검 합성 이미지에서 saturated_ratio≈0.5, dark_ratio≈0.5, sharpness.full>0, uniformity.grid 길이 9.
- test_pipeline: "sfr_test.bmp" 이름으로 저장한 합성 이미지 → chart_type sfr_iso12233, method rule, result.json 존재, 2회 실행 수치 동일.

완료 조건: pytest 통과, chart-agent analyze <<SAMPLE_DIR>>\<샘플> 이 result.json 을 만든다.
```

---

## 04 작업 B3: DLL 어댑터·mock·판정·보고서 (2:30~4:00)

```text
[작업 B3: DLL 어댑터, mock, 판정, 보고서] P0

파일: src/chart_agent/dll/base.py, dll/adapters/sfr.py, dll/adapters/mock_sfr.py, dll/__init__.py, judge.py, report.py,
tests/test_dll.py, tests/test_report.py. pipeline.py 의 스텁을 실제 호출로 바꿔 전체를 다시 출력하라.

dll/base.py
  class DllAdapter(Protocol):
      def load(self, dll_path: str | None) -> None: ...
      def call(self, function: str, native: np.ndarray, info: ImageInfo, params: dict) -> dict[str, float | int | str]: ...
  def roi_from(params, info) -> (x, y, w, h): params["roi"] 가 "center" 면 가로세로 각 50% 중앙 정렬,
      dict {x,y,w,h} 면 그대로(0<v<=1 이면 비율), "full" 이면 전체.
  def load_adapter(spec: str) -> DllAdapter: "모듈:클래스" 문자열 import.

dll/adapters/sfr.py — ctypes 어댑터. C 시그니처는 다음이다:
  <<DLL_SIGNATURE>>
  입력 종류: <<DLL_INPUT_KIND>>  (buffer 면 np.ascontiguousarray(native, dtype=np.uint16) 의 포인터를, path 면 info.path 를 c_char_p 로 넘긴다)
  출력 이름 순서: <<DLL_OUTPUTS>>  (double 배열로 받아 이 이름으로 dict 를 만든다)
  argtypes/restype 을 반드시 지정한다. 반환 코드가 0 이 아니면 ChartAgentError("E_DLL_CALL", f"{function} rc={rc}", "dll").
  로드 실패(OSError)는 E_DLL_LOAD 로 감싸고 WinError 193/126 해석 문구를 메시지에 넣는다.
  rgb8 입력(bmp/jpg)일 때는 gray 를 uint16 으로 올려(값 그대로) 넘긴다.

dll/adapters/mock_sfr.py — MockSfrAdapter. load 는 no-op. call 은 ROI 의 gray 라플라시안 분산 v 로
  mtf50 = round(min(0.9, v / 2000), 4), mtf30 = round(min(0.95, mtf50 * 1.5), 4) 를 반환한다(결정적 pseudo 값).
  params 에 "fail": true 가 있으면 RuntimeError 를 던진다(테스트용).

dll/__init__.py — run_project(img, project, registry) -> ProjectResult
  kind 가 mock 이면 mock_adapter, c 면 adapter 를 로드. 프로젝트의 analyses 를 순서대로 call 하고 결과 키를 "<analysis_id>.<key>" 로
  평탄화해 results 에 모은다. 예외는 dll_status failed, error 에 "코드: 메시지", 부분 결과는 유지. elapsed_ms 기록.

judge.py — judge(result_basic: dict, project: ProjectResult, registry, project_cfg) -> list[Judgement]
  (a) project analyses 의 thresholds: metric "<analysis_id>.<key>", value 는 results 에서, 없으면 value "n/a", passed None.
  (b) registry.basic_thresholds: metric 은 점 경로("basic.clipping.saturated_ratio") 로 result_basic 에서 찾는다.
  rule 평가: ">= x", "> x", "<= x", "< x", "== x", "between a b". 파싱 실패는 ValueError.

report.py — write_report(result: AnalysisResult, narrative: dict | None, out_dir) -> str(report_path)
  report.md 구성(한국어):
    # 차트 분석 보고서 — <파일명>
    > 수치 정본은 result.json 이다. 아래 서술은 표의 값만 인용한다.
    ## 개요: 파일, 차트 유형(판별 방법·신뢰도), 과제, 상태, 생성 시각
    ## 기본 평가: 표(항목 | 키 | 값)
    ## 과제 결과: 표(키 | 값) 또는 "dll_status: <값>, <error>"
    ## 판정: 표(항목 | 값 | 기준 | 결과: 합격/불합격/참고)
    ## 요약 / ## 해석 / ## 권고: narrative 의 summary/interpretation/recommendation. narrative 가 None 이면
       "판정 표를 참조하라. 불합격 N건, 참고 M건." 템플릿 문장.
    ## 문제: errors 목록 또는 "없음"
  result.report_path 를 갱신하고 result.json 을 다시 저장한다.

pipeline.py — run_project 와 judge, write_report(narrative=None) 를 연결한 전체 파일을 다시 출력한다.

tests
- test_dll: mock 으로 results 에 proj_sfr.center.mtf50 존재, params fail 로 dll_status failed 와 서버 생존(예외 전파 없음).
- test_report: 보고서의 판정 표 값이 result.json 의 judgement value 와 일치하는지 정규식으로 파싱해 비교.

완료 조건: pytest 통과. chart-agent analyze sfr 샘플 → report.md 생성, 판정 표 포함.
```

---

## 05 작업 B4: MCP 서버·호스트 등록 (4:00~5:30)

```text
[작업 B4: MCP 서버] P0

파일: src/chart_agent/mcp_server.py, scripts/host_mcp_config.json, tests/test_mcp_tools.py.
mcp 패키지의 FastMCP 를 쓴다: from mcp.server.fastmcp import FastMCP, Image. stdio 로 실행: python -m chart_agent.mcp_server.
환경 변수 CHART_AGENT_REGISTRY, CHART_AGENT_OUT 를 읽는다. 레지스트리는 서버 시작 시 1회 로드하되 analyze_image 호출마다
파일 mtime 이 바뀌었으면 다시 로드한다(시연 중 레지스트리 수정 반영).

도구 5개. 반환은 JSON 문자열(ensure_ascii=False, indent=1). docstring 은 아래 설명문을 토씨 그대로 넣는다.

1) list_inputs(path: str) -> str
   설명문: 폴더 또는 파일 경로를 받아 분석 가능한 파일(.bmp .jpg .jpeg .raw) 목록을 반환한다. 각 항목은 path, ext, size_bytes, has_sidecar다. 폴더를 분석할 때 가장 먼저 호출한다.

2) get_chart_catalog() -> str
   설명문: 레지스트리에 등록된 차트 유형(id, name, description), 과제(id, name, chart_types), 분석 항목과 임계값을 반환한다. 차트 유형이 unknown일 때 유형을 고르는 기준이며, 사용자가 등록된 분석을 물을 때도 쓴다.

3) analyze_image(path: str, chart_type: str | None = None, project_id: str | None = None,
                 width: int | None = None, height: int | None = None, bit_depth: int | None = None, bayer: str | None = None) -> str
   설명문: 이미지 1장을 로드 → 차트 판별(규칙) → 전체 이미지 기본 평가 → 과제 DLL 계산 → 임계값 판정까지 한 번에 실행하고 result JSON을 반환한다. raw는 width/height/bit_depth/bayer를 주지 않으면 사이드카 → 레지스트리 기본값 → 자동 추정 순으로 결정한다. 반환된 image_id는 get_preview와 write_report에 쓴다. classification.chart_type이 "unknown"이면 get_preview로 확인하고 chart_type을 지정해 다시 호출하라. input.raw_meta.needs_input이 true면 candidates를 사용자에게 확인한 뒤 인자를 지정해 다시 호출하라. 수치는 이 결과의 값만 인용하라.
   검증: chart_type·project_id·bayer 가 레지스트리/허용값에 없으면 {"error": "...", "allowed": [...]} 를 반환한다(예외를 던지지 않는다).
   반환 JSON 에서 basic.uniformity.grid 는 유지하되 4KB 를 넘기면 basic.stats 의 std 들을 제거한다.

4) get_preview(image_id: str) -> Image
   설명문: analyze_image가 만든 1024px 프리뷰 이미지를 반환한다. 차트 유형을 눈으로 판별하거나 사용자에게 보여줄 때 쓴다.
   out/<image_id>/preview.jpg 가 없으면 {"error": "unknown image_id"} 문자열을 반환한다.

5) write_report(image_id: str, summary: str, interpretation: str, recommendation: str) -> str
   설명문: image_id의 수치 표·판정 표에 전달받은 summary, interpretation, recommendation을 합쳐 report.md를 저장하고 경로를 반환한다. 세 문장 묶음은 한국어로 쓰고 judgement 표의 수치만 인용한다. 분석 1장마다 마지막에 호출한다.
   out/<image_id>/result.json 을 읽어 AnalysisResult 로 복원한 뒤 report.write_report 를 호출한다. 반환 {"report_path", "status"}.

scripts/host_mcp_config.json — 호스트 설정 예:
  {"mcpServers": {"chart-agent": {"command": "<<PYTHON_EXE>>", "args": ["-m", "chart_agent.mcp_server"],
   "env": {"CHART_AGENT_REGISTRY": "<<WORK_DIR>>\\config\\registry.yaml", "CHART_AGENT_OUT": "<<WORK_DIR>>\\out"}}}}

tests/test_mcp_tools.py — 서버를 띄우지 않고 도구 함수를 직접 import 해 호출: analyze_image 로 image_id 획득 → write_report → report_path 존재.
  잘못된 chart_type 은 error 와 allowed 반환.

완료 조건: pytest 통과. 호스트 설정 파일에 등록 후 호스트를 재시작하고 "chart-agent 도구 목록 보여줘"에 5개가 보인다.
```

---

## 06 작업 B5: raw 추정·판별 보강 (6:30~8:00)

```text
[작업 B5: raw 메타 추정과 판별 보강] P1

파일: src/chart_agent/loaders/raw_estimate.py, tests/test_raw_estimate.py. loaders/raw.py 의 resolve_raw_meta 와 pipeline.py 를
추정이 연결되도록 전체 다시 출력.

raw_estimate.py — estimate_raw_meta(path, known_resolutions: list[tuple[int,int]], default_bayer: str | None) -> RawMeta
  1. S = 파일 크기. 바이트/픽셀 b ∈ {2, 1} 에 대해 N = S / b 가 정수인 경우만 후보.
  2. w*h == N 인 (w,h) 를 known_resolutions + 기본 표(640x480, 1280x720, 1280x960, 1920x1080, 2592x1944, 3264x2448,
     4000x3000, 4032x3024, 4608x3456, 8000x6000) 에서 찾는다. 없으면 w % 8 == 0 이고 종횡비가 4:3, 3:2, 16:9, 1:1 중 하나(오차 2%)인
     약수 쌍을 후보로 만든다.
  3. 각 후보를 (h, w) 로 reshape 하고, 32행 간격으로 샘플한 행 r 과 r+2 의 평균 절대 차이를 점수로 삼는다(낮을수록 좋음).
     최저 점수 후보를 고르고 confidence = clamp(1 - best/second_best, 0, 1). 후보가 하나면 0.9.
  4. 비트 깊이: 최댓값이 들어가는 최소 비트를 {8,10,12,14,16} 에서 고른다. b==2 인데 최댓값 ≤ 255 면 8과 16 둘 다 후보.
  5. 베이어: 4개 위상 평면 평균 중 가장 가까운 두 평면을 G 로 본다. G 가 (0,1),(1,0) 이면 RGGB|BGGR, (0,0),(1,1) 이면 GRBG|GBRG.
     R/B 순서는 default_bayer 가 후보 안에 있으면 그것, 없으면 첫 번째를 쓰고 둘 다 candidates 에 넣는다.
  6. estimated=True. confidence < 0.8 또는 후보 2개 이상이면 needs_input=True. candidates 는 점수 오름차순 최대 5개.

resolve_raw_meta 는 override/사이드카/기본값이 모두 없을 때 estimate_raw_meta 를 호출한다. known_resolutions 는 레지스트리의
모든 projects[*].raw_defaults 에서 모은다. 추정을 썼으면 pipeline 의 errors 에 {"stage":"load","code":"E_RAW_ESTIMATE_AMBIGUOUS",...} 를
needs_input 일 때만 추가하고 상태를 partial 로 둔다.

classify.py 보강: unknown 이면 ChartClassification.needs_classification=True 이고, analyze_image 의 반환 JSON 최상위에
"next_action": "chart_type 이 unknown 이다. get_preview(image_id) 로 확인하고 chart_type 을 지정해 다시 호출하라." 를 넣는다.
needs_input 이면 "next_action": "raw 사양이 불확실하다. candidates 를 사용자에게 확인하고 width/height/bit_depth/bayer 를 지정해 다시 호출하라."

tests: 160x120 RGGB 10비트 합성 raw(사이드카 없음) → 추정 width/height/bit_depth 일치, bayer 가 RGGB 또는 BGGR.

완료 조건: pytest 통과. 사이드카 없는 샘플 raw 로 chart-agent analyze 실행 시 raw_meta.estimated true.
```

---

## 07 작업 B6: 일괄 처리·오류 경로·로그 (8:00~9:30)

```text
[작업 B6: 폴더 일괄, 오류 경로, 로그] P1

1. cli analyze 가 폴더를 받으면 지원 파일을 전부 처리하고 out/summary.json 에 [{path, image_id, status, chart_type, fail_count}] 를 쓴다.
   mcp list_inputs 도 같은 열거 함수를 쓴다.
2. run.log: out/<image_id>/run.log 에 JSON Lines 로 {ts, stage, elapsed_ms, code?, message?} 를 남긴다. mcp 도구 호출은
   out/mcp.log 에 {ts, tool, args(이미지 바이트 없음), elapsed_ms, ok} 를 남긴다.
3. 오류 경로 테스트 추가: (a) 크기 불일치 raw → status failed, errors 에 E_RAW_SIZE_MISMATCH 와 기대 크기
   (b) mock params fail → partial, dll_status failed, 기본 평가는 존재 (c) 없는 chart_type 으로 analyze_image → error+allowed.
4. faulthandler.enable() 을 mcp_server 와 cli 시작 시 켠다.
5. report.py 템플릿 문장을 다듬는다: 불합격 항목이 있으면 "불합격: <metric>=<value> (기준 <rule>)" 를 항목별 한 줄로.
6. 보고서 수치 일치 테스트(B3 test_report)를 기본 평가 표까지 확장한다.

변경 파일 전체를 다시 출력하라. 완료 조건: pytest 전체 통과, chart-agent analyze <<SAMPLE_DIR>> 로 summary.json 생성.
```

---

## 08 작업 B7: README·시연 준비 (9:30~10:30)

```text
[작업 B7: README 와 시연 준비] P1

1. README.md: 설치(setup.ps1), 레지스트리 설명과 과제 추가 방법(YAML 한 블록 + 어댑터 1파일), CLI 사용법,
   호스트 등록(scripts/host_mcp_config.json 을 <<HOST_MCP_CONFIG_PATH>> 에 병합), 시스템 프롬프트 투입 위치,
   시연 순서, 문제 해결(WinError 193/126, 도구 안 보임 → 호스트 재시작, raw 추정 틀림 → 인자 지정).
2. demo/ 폴더 구성 안내: sfr_01.bmp, sfr_02.jpg, flat_01.bmp, unknown_01.jpg, sensor_01.raw(+사이드카 없이) 와 각 파일의 기대 결과 표.
3. scripts/rehearsal_checklist.md: 서버 기동 → 도구 목록 → 시연 대사 6개 → 각 단계 소요 시간 기록란.

완료 조건: README 만 읽고 다른 사람이 설치·등록·시연을 재현할 수 있다.
```

---

## 20 런타임: 호스트 시스템 프롬프트 (프로젝트 지침 자리에 그대로)

```text
# 역할
당신은 카메라 화질 평가용 차트 이미지 분석 에이전트다. chart-agent 도구 서버의 도구만 사용해 분석하고,
수치는 절대 직접 계산하거나 추정하지 않는다.

# 절차 (이미지 1장마다 이 순서를 지킨다)
1. 경로가 폴더이면 list_inputs(path)로 파일 목록을 얻고 파일마다 2~5를 반복한다.
2. analyze_image(path)를 호출한다. 사용자가 차트 유형·과제·raw 사양을 말했으면
   chart_type, project_id, width, height, bit_depth, bayer 인자를 함께 넘긴다.
3. 결과를 확인하고 필요한 경우에만 다음을 수행한다. 결과에 next_action 이 있으면 그 지시를 따른다.
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
- 도구 오류는 숨기지 않고 사용자에게 알린다. 도구가 error 와 allowed 를 돌려주면 allowed 안에서 고쳐 다시 호출한다.
- 도구는 한 번에 하나만 호출하고 결과를 확인한 뒤 다음을 호출한다.
- 사용자가 chart_type 또는 project_id 를 지정하면 판별 단계를 생략한다.
- 도구 결과에 없는 분석을 요구받으면 "등록되지 않은 분석"이라고 답하고 get_chart_catalog 의 목록을 보여준다.
- 응답과 보고서는 한국어로 쓴다.
```

## 21 런타임: 도구 설명문

B4 프롬프트 안에 토씨 그대로 들어 있다. 코드 docstring이 곧 모델이 읽는 프롬프트이므로 임의로 줄이지 않는다.

## 22 런타임: 호스트 설정 JSON

B4가 만드는 `scripts/host_mcp_config.json`을 `<<HOST_MCP_CONFIG_PATH>>`에 병합하고 호스트를 재시작한다. 등록 확인 대사: "chart-agent 도구 목록 보여줘".

## 23 대체 경로 B: 호스트가 MCP 없이 로컬 명령 실행만 될 때의 시스템 프롬프트

```text
# 역할
당신은 카메라 화질 평가용 차트 이미지 분석 에이전트다. 분석은 반드시 아래 명령으로 수행하고 수치는 직접 계산하지 않는다.

# 명령
- 분석: <<PYTHON_EXE>> -m chart_agent.cli analyze "<경로>" [--chart-type ID] [--project ID] [--width N --height N --bit-depth N --bayer RGGB]
  결과: out/<image_id>/result.json, preview.jpg, report.md(템플릿 서술). 명령 출력 첫 줄에 result.json 경로가 나온다.
- 서술 붙이기: 서술 3절을 narrative.md 로 저장한 뒤 <<PYTHON_EXE>> -m chart_agent.cli attach-narrative <image_id> narrative.md
- 카탈로그: <<PYTHON_EXE>> -m chart_agent.cli catalog

# 절차
1. analyze 명령을 실행하고 result.json 을 읽는다.
2. classification.chart_type 이 unknown 이면 preview.jpg 를 열어 보고 카탈로그에서 유형을 고른 뒤 --chart-type 으로 다시 실행한다.
   input.raw_meta.needs_input 이 true 면 candidates 를 사용자에게 확인하고 인자를 지정해 다시 실행한다.
3. judgement 를 근거로 "## 요약", "## 해석", "## 권고" 세 절을 한국어로 narrative.md 에 쓴다. 표의 값만 인용한다.
4. attach-narrative 를 실행하고 report.md 경로, 차트 유형, 불합격 건수를 5줄 이내로 답한다.
```

이 경로를 쓰려면 B6에서 `attach-narrative`와 `catalog` 명령을 추가한다. 프롬프트: "cli 에 attach-narrative <image_id> <narrative.md>(## 요약/## 해석/## 권고 절을 파싱해 write_report 호출)와 catalog(registry.catalog_dict 를 JSON 출력) 명령을 추가하고 cli.py 전체를 다시 출력하라."

## 24 대체 경로 C: 채팅만 될 때의 해석 요청 템플릿

CLI가 `out/<image_id>/interpret_prompt.md`로 생성하게 하고, 사용자가 채팅에 붙여넣는다. 답을 `narrative.md`로 저장해 `attach-narrative`로 합친다.

```text
당신은 카메라 화질 평가 보고서 작성자다. 아래 표의 값만 인용해 한국어로 세 절을 쓴다. 새 수치를 만들거나 환산하지 않는다.
출력 형식은 정확히 "## 요약", "## 해석", "## 권고" 세 절이다. 요약은 3문장 이내, 해석은 불합격·참고 항목 중심으로 값과 기준을 그대로 인용,
권고는 재촬영·초점·노출·조명 조정 등 다음 행동과 근거 항목.

파일: {path}  차트 유형: {chart_type} (판별 {method}, 신뢰도 {confidence})  과제: {project_id}  상태: {status}

[기본 평가]
{basic_table}

[과제 결과]
{project_table}

[판정]
{judgement_table}

[문제]
{errors}
```

---

## 30 시연 대사

| 순서 | 사용자 대사 | 기대 동작 |
|---|---|---|
| 1 | chart-agent 도구 목록 보여줘 | 5개 도구 이름과 설명 |
| 2 | <<SAMPLE_DIR>>\sfr_01.bmp 분석해줘 | analyze_image → write_report, 경로·판정 요약 5줄 |
| 3 | <<SAMPLE_DIR>>\unknown_01.jpg 분석해줘 | unknown → get_preview → 후보 제시·질문 → 지정 후 재분석 |
| 4 | <<SAMPLE_DIR>>\sensor_01.raw 분석해줘 | 추정 후보 제시 → 사양 확인 → 분석 |
| 5 | 방금 보고서에서 불합격 항목을 왜 그런지 설명해줘 | judgement 값·기준 인용 해석 |
| 6 | (레지스트리에 flat_field 임계값 추가 후) <<SAMPLE_DIR>>\flat_01.bmp 분석해줘 | 코드 수정 없이 판정 반영 |

시연 전 리허설에서 2번 대사의 전체 소요 시간을 재고 2분을 넘으면 `basic.stats`의 std 항목을 반환 JSON에서 빼 결과를 줄인다.

## 31 문제 해결 프롬프트 (사내 LLM에 붙여넣기)

| 증상 | 프롬프트 |
|---|---|
| 호스트에 도구가 안 보임 | "MCP stdio 서버가 호스트에 안 보인다. scripts/host_mcp_config.json 의 command 경로가 절대 경로인지, python -m chart_agent.mcp_server 를 터미널에서 직접 실행했을 때 에러 없이 대기하는지, 패키지가 pip install -e . 로 설치됐는지 확인하는 점검 스크립트 scripts/check_mcp.py 를 작성하라. 서버 stderr 를 파일로 남기는 옵션도 추가하라." |
| WinError 193 | "DLL 이 32비트다. 32비트 Python 경로를 레지스트리 dll.python_exe 로 받아, 어댑터 호출을 subprocess 로 그 인터프리터에서 실행하고 JSON 으로 결과를 받는 dll/worker.py 와 runner 를 작성하라. 어댑터 인터페이스는 그대로 둔다." |
| WinError 126 | "DLL 의존성 누락이다. os.add_dll_directory 로 DLL 폴더를 검색 경로에 추가하고, 실패 시 dumpbin /dependents 대신 pefile 없이 PE import table 을 읽어 누락 DLL 이름을 출력하는 코드를 check_dll.py 에 추가하라." |
| raw 추정이 틀림 | "raw_estimate 의 행 상관 점수가 후보를 잘못 고른다. 점수에 열 방향(같은 위상 열 c, c+2) 차이를 더하고, known_resolutions 후보에 가중치 0.5 를 곱해 우선하도록 수정하라. 테스트에 실제 샘플 크기 {S 바이트} 케이스를 추가하라." |
| LLM이 수치를 지어냄 | 시스템 프롬프트 규칙 첫 줄을 "모든 수치는 도구 결과의 값을 복사한다. 복사할 수 없는 수치는 쓰지 않는다."로 바꾸고, write_report 설명문 끝에 "표에 없는 숫자가 포함되면 보고서가 거부된다"를 추가한다. 서버 쪽: "write_report 에서 세 서술에 포함된 소수·정수를 추출해 judgement/basic 값 집합에 없으면 {"error":"unknown number", "numbers":[...]} 를 반환하게 수정하라." |
| 도구를 여러 개 동시에 호출 | 시스템 프롬프트에 이미 "한 번에 하나" 규칙이 있다. 서버는 순서를 가정하지 않으므로 동작엔 문제 없다. 결과 혼동이 보이면 "각 도구 결과의 image_id 를 응답에 명시하라"를 절차 5에 추가한다. |
| 결과 JSON이 너무 길어 느림 | "analyze_image 반환에서 basic.uniformity.grid 와 모든 *_std 를 제거하고 preview_path, errors, next_action 은 유지하도록 수정하라. 전체 result.json 은 파일에 그대로 둔다." |
| 호스트가 프리뷰 이미지를 못 봄 | 시스템 프롬프트 절차 3의 unknown 처리를 "preview_path 를 사용자에게 알려 주고 사용자가 이미지를 채팅에 첨부하면 유형을 고른다"로 바꾼다. |

## 32 자체 점검 프롬프트 (코드 완성 후)

```text
[자체 점검]
아래 인수 조건마다 현재 코드가 만족하는지 "조건 | 근거 파일·함수 | 통과/미통과/미확인 | 미통과면 수정 제안" 표로 답하라. 추측하지 말고 코드에서 근거를 찾아라.
AC-01 bmp/jpg 로드 시 폭·높이·채널이 원본과 일치
AC-02 raw + 메타 지정 로드 성공, 크기 불일치면 E_RAW_SIZE_MISMATCH
AC-03 파일명 규칙으로 sfr 샘플은 sfr_iso12233, 규칙 없는 파일은 unknown
AC-04 proj_sfr 분석 시 results 에 proj_sfr.center.mtf50, dll_status ok
AC-05 DLL 예외 시 dll_status failed, 기본 평가 유지, 프로세스 생존
AC-06 CLI 단독 실행으로 result.json 과 템플릿 report.md 생성
AC-07 report.md 판정 표 수치 == result.json judgement 값
AC-08 레지스트리에 유형·과제 추가만으로 동작(코어 수정 없음)
AC-09 없는 chart_type/analysis_id 호출 시 error 와 allowed 반환
AC-10 같은 입력 2회 실행 시 수치 동일
AC-11 호스트 1회 지시로 analyze_image → write_report 호출, report.md 생성 (리허설 확인)
AC-12 사이드카 없는 raw 추정 폭·높이·비트 일치
미통과 항목이 있으면 수정 코드를 파일 전체로 출력하라.
```

## 33 컨텍스트 복구 프롬프트 (새 대화로 옮길 때)

`00 브리프`를 먼저 붙인 뒤 이어서:

```text
[진행 상황]
완료: {완료한 작업 번호와 파일 목록, 예: B0~B3, models.py registry.py loaders/* basic.py pipeline.py cli.py dll/* judge.py report.py, pytest 전부 통과}
현재 파일 중 당신이 알아야 할 것: {주요 파일 2~3개의 전체 내용을 붙여넣기. 최소한 models.py, registry.py, pipeline.py}
지금 할 작업: {다음 작업 번호}
위 파일과 호환되게 작성하라. 기존 함수 시그니처를 바꾸지 않는다.
```

## 변경 이력

| 일자 | 변경 | 근거 | 상태 |
|---|---|---|---|
| 2026-10-11 | 최초 작성 | 사용자 요청(해커톤에서 클로드 사용 불가, 사내 LLM용 프롬프트 사전 확정) | `awaiting-approval` |
| 2026-10-13 | 웹 개발로 목표 변경에 따라 [웹 프롬프트 팩](./hackathon-prompt-pack-web.md)으로 대체 | 사용자 결정 | `superseded` |
