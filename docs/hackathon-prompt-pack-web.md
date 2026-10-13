# DESIGN-image-analysis-agent-prompts-web: 웹 해커톤 프롬프트 팩

> 문서 유형: `design`
> 작업 ID: `20261011-image-analysis-agent`
> 상태: `awaiting-approval`
> 기준선: `draft`
> 작성일: 2026-10-13
> 최종 갱신: 2026-10-13
> 관련 문서: [DESIGN-image-analysis-agent: 설계](./design.md), [REQ-image-analysis-agent: 요구사항](./requirements.md), [이전 MCP 프롬프트 팩](./hackathon-prompt-pack.md)(대체됨)

## 요약

- 목적: 해커톤에서 클로드 없이 사내 LLM(GLM-5.3-Flash)만으로 **웹 앱 형태의 Agent**를 만들고 운용하기 위해, 붙여넣을 프롬프트 전부를 미리 확정한다. 사내 LLM은 설계 문맥을 모르므로 각 프롬프트가 혼자서도 동작하도록 사양을 안에 담았다.
- 현재 결론 또는 상태: 프롬프트 3묶음. **빌드용 00~10**은 사내 LLM에게 코드를 쓰게 하는 프롬프트(브리프 1 + 작업 B0~B9), **런타임용 20~24**는 백엔드가 LLM API에 보내는 판별·서술·채팅 프롬프트와 수동 대체 경로 템플릿, **운영용 30~33**은 시연 대사·문제 해결·자체 점검·컨텍스트 복구다. [설계](./design.md) 3절·5절·9절의 자기완결 확장판이다.
- 다음 행동: 해커톤 전에 [자리표시자](#자리표시자)를 채운다. 당일에는 `00 브리프`를 새 대화마다 먼저 붙여넣고 `01`부터 순서대로 쓴다.

## 문서 연결

| 방향 | 관계 | 대상 문서 | 대상 항목 | 비고 |
|---|---|---|---|---|
| input | baseline | [DESIGN-image-analysis-agent: 설계](./design.md) | 3.1~3.7, 5.2~5.6, 9.2, 9.3 | 이 문서는 설계의 프롬프트 부속이다 |
| input | baseline | [REQ-image-analysis-agent: 요구사항](./requirements.md) | FR-01~FR-13, AC-01~AC-15 | 자체 점검 프롬프트의 기준 |
| input | replacement | [DESIGN-image-analysis-agent-prompts: MCP 프롬프트 팩](./hackathon-prompt-pack.md) | document | 이 문서가 대체함 |

## 사용 방법

1. **해커톤 전:** 자리표시자 표를 채우고 이 문서를 반입한다. 휠 파일을 `wheels/`에 받아 둔다. 샘플 4종을 준비한다.
2. **당일 새 대화마다:** `00 프로젝트 브리프`를 첫 메시지로 붙여넣고 "브리프 확인" 답을 받은 뒤 작업 프롬프트를 준다. 대화가 길어져 느려지면 새 대화를 열고 `33 컨텍스트 복구`로 잇는다.
3. **빌드:** `01`→`10` 순서. 각 프롬프트 끝의 완료 조건을 확인하고 넘어간다. B0에서 LLM API가 확인되면 `07(B6)`을 `06(B5)` 앞으로 당긴다.
4. **6시간 체크포인트:** 브라우저에서 업로드→결과→보고서가 되면 P1로, 안 되면 P1을 버리고 P0 안정화.
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
| `<<DLL_INPUT_KIND>>` | `buffer`(픽셀 포인터) 또는 `path`(파일 경로) | `buffer` |
| `<<RAW_W>> <<RAW_H>> <<RAW_BITS>> <<RAW_BAYER>>` | 과제 raw 기본 사양 | `4000 3000 10 RGGB` |
| `<<SAMPLE_DIR>>` | 샘플 이미지 폴더 | `C:\hack\samples` |
| `<<LLM_BASE_URL>>` | 사내 LLM API 주소(OpenAI 호환, `/v1`까지) | `https://llm.company.local/v1` |
| `<<LLM_MODEL>>` | 모델 ID | `glm-5.3-flash` |

DLL·LLM 정보를 해커톤 전에 못 구하면 예시 값으로 두고 mock DLL·manual 모드로 진행한다. 둘 다 나중에 파일 1개와 환경 변수만 바꾸면 된다.

---

## 00 프로젝트 브리프 (매 새 대화의 첫 메시지)

```text
[프로젝트 브리프 — 이 대화 내내 아래를 기준으로 답하라]

당신은 Python·웹 시니어 엔지니어다. 나는 12시간 해커톤에서 "카메라 화질 평가용 차트 이미지 분석 AI Agent"를 웹 앱으로 만든다.
당신의 역할은 내가 주는 작업 프롬프트마다 완전한 코드를 쓰는 것이다. 설명은 짧게, 코드는 파일 전체를 출력한다.

1. 목표
- 브라우저에서 이미지(bmp/jpg/raw)를 올리면 백엔드가 차트 판별 → 전체 이미지 기본 평가 → 과제 DLL(SFR 해상도) 계산 →
  임계값 판정 → 보고서(report.md)까지 자동으로 끝낸다. 차트 유형이나 raw 사양이 불확실하면 작업을 멈추고 화면에서 사용자에게 확인한다.
- LLM(사내 GLM-5.3-Flash, OpenAI 호환 API)은 세 곳에서만 쓴다: (1) 규칙 판별 실패 시 프리뷰로 후보 제시, (2) 보고서 서술(요약·해석·권고),
  (3) 결과 화면 채팅. API가 없으면 (1)은 사용자 선택, (2)는 "해석 프롬프트 복사 → 사내 채팅에 붙여넣기 → 답을 화면에 붙여넣기"로 대체된다.
- 같은 코어를 CLI로도 실행한다. 수치 계산은 전부 결정적 코드이며 코드 안에서 LLM이 수치를 만들지 않는다.
- 우선순위: P0 = 시연 필수, P1 = 시간이 남으면. 내가 P0라고 하면 P1 기능을 넣지 않는다.

2. 환경과 제약
- Windows 11, Python 3.11. 허용 패키지: fastapi, uvicorn, python-multipart, openai, httpx, numpy, pillow, pydantic(v2), pyyaml,
  opencv-python, pytest. 다른 패키지·프레임워크 금지. 프런트엔드는 정적 HTML/CSS/JS 3파일, 외부 CDN·빌드 도구 금지.
- 과제 DLL은 C 함수(ctypes). 실제 DLL이 없으면 같은 호출 시그니처의 mock(Python)을 쓴다.
- raw는 헤더 없는 베이어 덤프. 16비트 little-endian unpacked(픽셀당 2바이트), 유효 비트 8/10/12/14/16, 패턴 RGGB/BGGR/GRBG/GBRG.
- 서버는 127.0.0.1:8000. 인증 없음. 업로드 상한 200MB. 결과는 out/<job_id>/ 에 파일로 저장.

3. 디렉터리 (src 레이아웃, 패키지명 chart_agent, pyproject.toml, 콘솔 스크립트 chart-agent = chart_agent.cli:main)
  <<WORK_DIR>>/
    config/registry.yaml
    prompts/classify.md, narrative.md, chat.md, interpret_template.md
    src/chart_agent/__init__.py, models.py, errors.py, registry.py, pipeline.py, cli.py, basic.py, classify.py, judge.py, report.py
    src/chart_agent/loaders/__init__.py, image.py, raw.py, raw_estimate.py
    src/chart_agent/dll/__init__.py, base.py, adapters/__init__.py, adapters/sfr.py, adapters/mock_sfr.py
    src/chart_agent/llm/__init__.py, client.py, mock.py, probe.py, prompts.py
    src/chart_agent/web/__init__.py, app.py, jobs.py, api.py, chat.py
    static/index.html, app.js, style.css
    scripts/, tests/, demo/, out/

4. 데이터 모델 (src/chart_agent/models.py) — 그대로 쓴다
  class RawMeta(BaseModel):
      width: int; height: int; bit_depth: int
      bayer: Literal["RGGB","BGGR","GRBG","GBRG"]
      packing: Literal["unpacked16","unpacked8"] = "unpacked16"
      estimated: bool = False; confidence: float = 1.0
      candidates: list[dict] = []; needs_input: bool = False
  class RawOverride(BaseModel):
      width: int | None = None; height: int | None = None; bit_depth: int | None = None; bayer: str | None = None
  class ImageInfo(BaseModel):
      path: str; format: Literal["bmp","jpg","raw"]; width: int; height: int; channels: int; bit_depth: int
      sha256: str; raw_meta: RawMeta | None = None
  class Candidate(BaseModel):
      chart_type: str; confidence: float; reason: str = ""
  class ChartClassification(BaseModel):
      chart_type: str; confidence: float; method: Literal["user","rule","llm","none"]
      needs_classification: bool = False; candidates: list[Candidate] = []
  class ProjectResult(BaseModel):
      project_id: str | None; dll_status: Literal["ok","failed","skipped"]
      results: dict[str, float | int | str] = {}; error: str | None = None; elapsed_ms: int | None = None
  class Judgement(BaseModel):
      metric: str; value: float | int | str; rule: str | None; passed: bool | None
  class Narrative(BaseModel):
      summary: str; interpretation: str; recommendation: str
  class AnalysisResult(BaseModel):
      schema_version: Literal["1"] = "1"; image_id: str; input: ImageInfo; classification: ChartClassification
      basic: dict[str, dict]; project: ProjectResult; judgement: list[Judgement]
      status: Literal["ok","partial","failed"]; errors: list[dict] = []
      preview_path: str; report_path: str | None = None
      narrative_source: Literal["llm","template","manual"] | None = None
  class Job(BaseModel):
      job_id: str; created_at: str; filename: str
      state: Literal["queued","running","needs_input","done","partial","failed"]
      llm_mode: Literal["auto","manual","off"]
      steps: list[dict] = []            # {"ts","stage","message","elapsed_ms"}
      needs: dict | None = None         # {"type":"chart_type"|"raw_meta","candidates":[...],"preview_url":"..."}
      params: dict = {}; result: AnalysisResult | None = None; error: str | None = None
  상태 규칙: 모든 단계 성공 ok. unknown 차트, DLL 실패, raw 추정 사용, LLM 대체 사용 중 하나라도 있으면 partial. 로드 실패 failed.
  image_id = 파일 바이트 sha256 앞 12자리. job_id = uuid4 앞 8자리.

5. 레지스트리 (config/registry.yaml) — 아래 형식. registry.py 가 pydantic 으로 로드·검증한다
  schema_version: 1
  out_dir: ./out
  server: {host: 127.0.0.1, port: 8000, max_upload_mb: 200}
  llm:
    model: <<LLM_MODEL>>
    base_url_env: GLM_BASE_URL
    api_key_env: GLM_API_KEY
    confidence_threshold: 0.7
    classify:  {temperature: 0.3, reasoning_effort: low,    timeout_s: 60,  max_tokens: 400,  max_image_edge: 1024}
    narrative: {temperature: 0.7, reasoning_effort: medium, timeout_s: 120, max_tokens: 1200}
    chat:      {temperature: 0.5, reasoning_effort: low,    timeout_s: 90,  max_tokens: 800,  max_steps: 6}
    retries: 2
  pipeline: {isolate: none}
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
        kind: mock                       # c | mock
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
  코드: E_UNSUPPORTED_FORMAT, E_UPLOAD_TOO_LARGE, E_RAW_META_MISSING, E_RAW_SIZE_MISMATCH, E_RAW_ESTIMATE_AMBIGUOUS,
        E_DLL_LOAD, E_DLL_CALL, E_DLL_NOT_REGISTERED, E_UNKNOWN_ANALYSIS, E_UNKNOWN_CHART_TYPE, E_UNKNOWN_PROJECT,
        E_LLM_UNAVAILABLE, E_LLM_PARSE, E_JOB_NOT_FOUND, E_JOB_STATE
  API 오류 응답: {"error": {"code": ..., "message": ..., "allowed": [...]?}}

7. REST API (prefix /api) — 요약. 상세는 작업 B4 에서 준다
  GET /health, GET /catalog, POST /jobs(multipart), GET /jobs/{id}, POST /jobs/{id}/resume, GET /jobs/{id}/preview.jpg,
  GET /jobs/{id}/report.md, GET /jobs/{id}/interpret-prompt, POST /jobs/{id}/narrative, [P1] GET /jobs, POST /jobs/{id}/chat

8. 출력 규칙 (반드시 지켜라)
- 생성·변경하는 파일마다 "### 파일: <상대경로>" 제목 뒤에 파일 전체 내용을 하나의 코드 블록으로 출력한다. 일부만 출력하지 않는다.
- 마지막에 실행 명령과 테스트 명령을 적는다.
- 설명은 10줄 이내. 가정이 필요하면 코드 상단 주석에 적고 진행한다. 질문으로 멈추지 않는다.
- 함수는 작게, 타입 힌트, docstring 한 줄. 전역 가변 상태는 JobManager 안에만. 로그는 logging 모듈. 비밀 정보 없음.
- 같은 파일을 다시 요청받으면 이전 내용과 호환되게 전체를 다시 출력한다. 기존 함수 시그니처를 바꾸지 않는다.

이해했으면 "브리프 확인"이라고만 답하라.
```

---

## 01 작업 B0: 환경 확인과 LLM 탐침 (0:00~0:30)

```text
[작업 B0: 환경 확인] P0

다음 네 파일을 만들어라.

1. setup.ps1
- <<WORK_DIR>> 에서 python -m venv .venv 를 만들고 .venv\Scripts\python.exe -m pip install 로
  fastapi uvicorn python-multipart openai httpx numpy pillow pydantic pyyaml opencv-python pytest 를 설치한다.
- 변수 $Offline 이 $true 이면 --no-index --find-links .\wheels 를 붙인다. 마지막에 pip install -e . 를 실행한다.

2. scripts/check_dll.py
- 인자: DLL 경로, 함수명. (a) ctypes.WinDLL 로드 성공 여부 (b) 함수 존재 여부 (c) PE 헤더 Machine 필드로 32/64비트
  (d) 실패 시 WinError 해석: 193 = 비트 불일치, 126 = 의존 DLL 누락. 표로 출력, 실패면 종료 코드 1.

3. scripts/check_samples.py
- 인자: 폴더. .bmp/.jpg/.jpeg/.raw 파일마다 크기, 이미지면 해상도·모드, raw 면 "픽셀당 2바이트 가정 픽셀 수"와 일치하는 (w,h) 후보를
  표에서 찾아 출력: 640x480, 1280x720, 1280x960, 1920x1080, 2592x1944, 3264x2448, 4000x3000, 4032x3024, 4608x3456, 8000x6000.

4. scripts/probe_llm.py
- 환경 변수 GLM_BASE_URL, GLM_API_KEY(없으면 "none"), 인자 --model (기본 <<LLM_MODEL>>). openai 패키지의 OpenAI(base_url=..., api_key=...) 사용.
- 순서대로 시도하고 각 결과(ok/fail, 지연 ms, 오류 메시지 앞 200자)를 표와 out/probe.json 으로 남긴다. 하나가 실패해도 계속한다.
  P1 텍스트만: "답은 'pong' 한 단어" → 응답 포함 여부
  P2 이미지: 64x64 빨간 PNG 를 만들어 data:image/png;base64 로 image_url 첨부, "무슨 색인가" → 응답에 '빨' 또는 'red'
  P3 response_format={"type":"json_schema","json_schema":{"name":"t","strict":true,"schema":{"type":"object","properties":{"a":{"type":"integer"}},"required":["a"],"additionalProperties":false}}}
  P4 response_format={"type":"json_object"}
  P5 프롬프트에 "JSON 만 출력: {\"a\": 1}" → 응답에서 첫 { ... } 블록 파싱
  P6 tools=[{"type":"function","function":{"name":"ping","description":"ping","parameters":{"type":"object","properties":{}}}}], tool_choice="auto",
     "ping 도구를 호출하라" → tool_calls 존재 여부
  P7 extra_body={"reasoning_effort":"low"} 와 {"thinking":{"type":"enabled"}} 각각 → 400 여부와 지연
- 마지막 줄에 권장 모드를 출력: P1 실패 → "manual", P1 성공·P2 성공 → "auto", P2 실패 → "auto(텍스트만, 판별 보조 불가)".

완료 조건: 네 파일이 실행되고 사용법이 docstring 에 있다. 실행 명령도 적어라.
```

---

## 02 작업 B1: 모델·레지스트리·로더 (0:30~1:30)

```text
[작업 B1: 모델, 레지스트리, 로더] P0

브리프 4절 모델, 5절 레지스트리, 6절 오류를 그대로 구현하라. 파일: pyproject.toml, src/chart_agent/__init__.py, models.py, errors.py,
registry.py, loaders/__init__.py, loaders/image.py, loaders/raw.py, config/registry.yaml(브리프 5절), tests/conftest.py, tests/test_loaders.py.

registry.py
- Registry.load(path=None) -> Registry. path 가 None 이면 환경 변수 CHART_AGENT_REGISTRY, 없으면 config/registry.yaml.
- 검증: 중복 id, 없는 project 참조, 임계값 문법, dll.kind 값. 실패는 ValueError 메시지.
- 조회: chart_type(id), project(id), project_for_chart(chart_id), analyses_for(project_id), all_analysis_ids(), chart_type_ids(), project_ids(),
  catalog_dict() -> {"chart_types":[{id,name,description,project}], "projects":[{id,name,analyses:[{id,thresholds}]}], "basic_thresholds":{...}},
  llm_settings(), server_settings(), out_dir(절대 경로), file_hash().

loaders/image.py — load_bmp_jpg(path) -> (rgb8 (H,W,3) uint8, ImageInfo). Pillow convert("RGB"), 원본 mode L 이면 channels=1, bit_depth 8.

loaders/raw.py
- resolve_raw_meta(path, override: RawOverride|None, project_defaults: dict|None) -> RawMeta|None
  우선순위: override(전부 있으면) > 사이드카 "<path>.raw.json" > project_defaults > None. override 가 일부면 나머지를 뒤 순위로 채운다.
- load_raw(path, meta) -> (bayer_u16 (H,W), rgb8 (H,W,3), ImageInfo). 기대 크기 = w*h*2(unpacked16) 또는 w*h(unpacked8).
  불일치면 ChartAgentError("E_RAW_SIZE_MISMATCH", f"expected {expected} bytes, got {actual}", "load").
  np.fromfile(dtype="<u2") → reshape → OpenCV 디모자이크. 패턴 상수 매핑은 반드시 다음(OpenCV 명명이 반대다):
    RGGB -> cv2.COLOR_BayerBG2RGB, BGGR -> cv2.COLOR_BayerRG2RGB, GRBG -> cv2.COLOR_BayerGB2RGB, GBRG -> cv2.COLOR_BayerGR2RGB
  uint16 그대로 cvtColor 하고 (2**bit_depth - 1) 로 나눠 0..255 uint8.
- make_preview(rgb8, out_path, long_edge=1024, stretch=False): LANCZOS 리사이즈, JPEG 품질 85. stretch=True 면 1~99 퍼센타일 대비 확장(raw 프리뷰 전용).

loaders/__init__.py
- @dataclass LoadedImage: native (raw 면 bayer u16, 아니면 rgb8), rgb8, info
- load_any(path, override, project_defaults) -> LoadedImage. 미지원 확장자 E_UNSUPPORTED_FORMAT. raw 인데 meta None 이면
  E_RAW_META_MISSING("width,height,bit_depth,bayer 가 필요하다"). sha256 는 파일 전체.

tests
- conftest: 64x48 RGB(왼쪽 반 흰, 오른쪽 반 검) bmp/jpg 픽스처, 32x24 RGGB 10비트 u16(R=800,G=500,B=200) raw 픽스처(tmp_path).
- test: bmp/jpg 크기·채널, raw rgb8 의 R>G>B, 사이드카 우선순위, 크기 불일치 예외 코드, 레지스트리 로드와 catalog_dict 키.

완료 조건: .venv\Scripts\python.exe -m pytest -q tests/test_loaders.py 통과.
```

---

## 03 작업 B2: 기본 평가·파이프라인·CLI (1:30~2:15)

```text
[작업 B2: 기본 평가, 판별 규칙, 파이프라인, CLI] P0

파일: src/chart_agent/basic.py, classify.py, pipeline.py, cli.py, tests/test_basic.py, tests/test_pipeline.py.

basic.py — run_basic(img: LoadedImage) -> dict[str, dict]. numpy/OpenCV 만, 결정적, 소수 round(x, 4).
  gray = cv2.cvtColor(rgb8, COLOR_RGB2GRAY)
  "basic.info": {width, height, channels, bit_depth, format}
  "basic.stats": {r_mean, r_std, g_mean, g_std, b_mean, b_std, min, max, luma_mean}
  "basic.clipping": {saturated_ratio: gray==255 비율, dark_ratio: gray==0 비율}
  "basic.sharpness": {center: 중앙 50%x50% ROI 라플라시안 분산, full: 전체}  cv2.Laplacian(gray, cv2.CV_64F).var()
  "basic.noise": {sigma: 1.4826 * median(|gray - medianBlur(gray,3)|)}
  "basic.uniformity": {grid: 3x3 블록 평균 9개(행 우선), corner_center_ratio: 네 모서리 평균 최소 / 중앙 평균}
  raw 면 "basic.bayer_stats": {r_mean, gr_mean, gb_mean, b_mean, r_std, gr_std, gb_std, b_std} (native u16, 패턴별 위상 매핑)

classify.py — classify_by_rules(path, user_chart_type, registry) -> ChartClassification
  user_chart_type 지정(없는 id 면 E_UNKNOWN_CHART_TYPE) → method "user", 1.0
  파일명 stem 에 filename_regex search → method "rule", 1.0 (첫 매치)
  아니면 "unknown", 0.0, method "none", needs_classification True, candidates 는 카탈로그 전체를 confidence 0 으로.

pipeline.py — run(path, chart_type=None, project_id=None, raw_override=None, registry=None, out_dir=None, llm=None, on_step=None) -> AnalysisResult
  1) 판별(규칙). 2) llm 이 있고 unknown 이면 llm.classify(preview, filename, catalog) 를 시도(B6 에서 구현, 지금은 llm 이 None 이면 건너뜀).
  3) project 결정: project_id 인자 > 차트 유형의 project. 4) load_any(raw_defaults 는 project 의 것).
  5) run_basic. 6) run_project (B3. 지금은 ProjectResult(project_id, "skipped") 스텁). 7) judge (B3. 지금은 []).
  8) out_dir/<image_id>/preview.jpg(raw 는 stretch=True), result.json 저장. 9) 상태 규칙.
  on_step(stage, message) 가 있으면 단계마다 호출. ChartAgentError 는 errors 에 넣고 계속, load 실패만 failed 로 종료.
  주의: 프리뷰는 로드 직후 만들어 needs_classification 이어도 preview_path 가 있게 한다.

cli.py — argparse. analyze <path> [--chart-type] [--project] [--width --height --bit-depth --bayer] [--out] [--registry],
  validate-registry, catalog(JSON 출력). 종료 코드 0/1/2. analyze 는 첫 줄에 result.json 경로, 둘째 줄에 status 출력.

tests
- test_basic: saturated_ratio≈0.5, dark_ratio≈0.5, sharpness.full>0, grid 길이 9.
- test_pipeline: "sfr_test.bmp" → sfr_iso12233/rule, result.json 존재, 2회 실행 수치 동일. "photo_test.jpg" → unknown, needs_classification True, preview 존재.

완료 조건: pytest 통과. chart-agent analyze <<SAMPLE_DIR>>\<sfr 샘플> 이 result.json 을 만든다.
```

---

## 04 작업 B3: DLL 어댑터·mock·판정·보고서·수동 서술 (2:15~3:30)

```text
[작업 B3: DLL, 판정, 보고서, 수동 서술] P0

파일: src/chart_agent/dll/base.py, dll/adapters/sfr.py, dll/adapters/mock_sfr.py, dll/__init__.py, judge.py, report.py,
prompts/interpret_template.md, tests/test_dll.py, tests/test_report.py. pipeline.py 와 cli.py 는 바뀐 부분을 반영해 전체 다시 출력.

dll/base.py
  class DllAdapter(Protocol):
      def load(self, dll_path: str | None) -> None: ...
      def call(self, function: str, native: np.ndarray, info: ImageInfo, params: dict) -> dict[str, float | int | str]: ...
  roi_from(params, info) -> (x,y,w,h): "center" = 50%x50% 중앙, dict {x,y,w,h}(0<v<=1 은 비율), "full".
  load_adapter(spec "모듈:클래스") -> DllAdapter

dll/adapters/sfr.py — ctypes 어댑터. C 시그니처: <<DLL_SIGNATURE>>
  입력 종류 <<DLL_INPUT_KIND>>: buffer 면 np.ascontiguousarray(native, dtype=np.uint16) 포인터, path 면 info.path 를 c_char_p.
  출력 이름 순서 <<DLL_OUTPUTS>> 를 double 배열로 받아 dict. argtypes/restype 지정. rc != 0 → E_DLL_CALL. OSError → E_DLL_LOAD(193/126 해석 포함).
  rgb8 입력이면 gray 를 uint16 으로 올려 넘긴다.

dll/adapters/mock_sfr.py — MockSfrAdapter. load 는 no-op. call 은 ROI gray 라플라시안 분산 v 로
  mtf50 = round(min(0.9, v/2000), 4), mtf30 = round(min(0.95, mtf50*1.5), 4). params["fail"] 가 true 면 RuntimeError(테스트용).

dll/__init__.py — run_project(img, project_cfg, registry) -> ProjectResult. kind 에 따라 어댑터 선택, analyses 순서대로 call,
  결과 키를 "<analysis_id>.<key>" 로 평탄화. 예외는 dll_status failed + error("코드: 메시지"), 부분 결과 유지, elapsed_ms.
  run_single_analysis(img, project_cfg, analysis_id, params, registry) -> dict 도 제공(채팅 도구용). 없는 id 는 E_UNKNOWN_ANALYSIS.

judge.py — judge(basic, project_result, project_cfg, registry) -> list[Judgement]. 과제 thresholds 와 basic_thresholds 평가.
  값 없으면 value "n/a", passed None. 문법 파싱 실패 ValueError.

report.py
- render_tables(result) -> str: "## 기본 평가" 표(항목|키|값), "## 과제 결과" 표 또는 dll_status 문구, "## 판정" 표(항목|값|기준|결과 합격/불합격/참고).
- write_report(result, narrative: Narrative|None, source: str|None, out_dir) -> report_path. report.md 구성(한국어):
    # 차트 분석 보고서 — <파일명>
    > 수치 정본은 result.json 이다. 서술은 표의 값만 인용한다. 서술 출처: llm | template | manual
    ## 개요(파일, 차트 유형·판별 방법·신뢰도, 과제, 상태, 생성 시각) + render_tables + ## 요약 / ## 해석 / ## 권고 + ## 문제(errors 또는 "없음")
  narrative 가 None 이면 템플릿: "판정 표를 참조하라. 불합격 N건, 참고 M건." 과 불합격 항목별 "불합격: <metric>=<value> (기준 <rule>)" 한 줄씩.
  result.report_path, narrative_source 갱신 후 result.json 재저장.
- build_interpret_prompt(result, registry) -> str: prompts/interpret_template.md 의 {path} {chart_type} {method} {confidence} {project_id}
  {status} {basic_table} {project_table} {judgement_table} {errors} 를 채운다.
- parse_narrative(text) -> Narrative: JSON 이면 그대로, 아니면 "## 요약" "## 해석" "## 권고" 절을 파싱. 둘 다 실패면 ValueError.

prompts/interpret_template.md — 내용은 다음 텍스트 그대로:
  당신은 카메라 화질 평가 보고서 작성자다. 아래 표의 값만 인용해 한국어로 세 절을 쓴다. 새 수치를 만들거나 환산하지 않는다.
  출력 형식은 정확히 "## 요약", "## 해석", "## 권고" 세 절이다. 요약은 3문장 이내(차트 유형, 전체 상태, 불합격 건수), 해석은 불합격·참고 항목 중심으로
  값과 기준을 그대로 인용, 권고는 재촬영·초점·노출·조명 조정 등 다음 행동과 근거 항목.
  파일: {path}  차트 유형: {chart_type} (판별 {method}, 신뢰도 {confidence})  과제: {project_id}  상태: {status}
  [기본 평가]
  {basic_table}
  [과제 결과]
  {project_table}
  [판정]
  {judgement_table}
  [문제]
  {errors}

pipeline.py — run_project, judge, write_report(narrative=None, source="template") 연결. cli.py — attach-narrative <image_id> <narrative.md> 추가
  (parse_narrative → write_report(source="manual")), analyze 뒤에 interpret_prompt.md 도 out/<image_id>/ 에 저장.

tests
- test_dll: mock 으로 proj_sfr.center.mtf50 존재, params fail → dll_status failed 이고 예외가 전파되지 않음.
- test_report: 판정 표 값 == result.json judgement value(정규식 파싱), parse_narrative 가 "## 요약/해석/권고" 텍스트와 JSON 둘 다 처리.

완료 조건: pytest 통과. chart-agent analyze sfr 샘플 → report.md, interpret_prompt.md 생성. attach-narrative 로 서술 반영.
```

---

## 05 작업 B4: 웹 백엔드 (3:30~4:30)

```text
[작업 B4: FastAPI 백엔드] P0

파일: src/chart_agent/web/__init__.py, web/app.py, web/jobs.py, web/api.py, tests/test_api.py. cli.py 에 serve 명령 추가(전체 다시 출력).

web/jobs.py — JobManager
- __init__(registry, llm=None). jobs: dict[str, Job]. out_dir/<job_id>/ 에 input/<원본명>, job.json 저장. 저장은 상태 변경마다.
- create(upload_filename, file_bytes, params: dict, use_llm: bool) -> Job: 확장자 검사(E_UNSUPPORTED_FORMAT), 크기 검사(E_UPLOAD_TOO_LARGE, server.max_upload_mb),
  파일 저장, state queued, llm_mode = "off" if not use_llm else ("auto" if llm else "manual").
- run(job_id): state running → pipeline.run(path, chart_type, project_id, RawOverride(**params), registry, out_dir/<job_id>, llm if auto, on_step=steps 기록)
  → 결과 판정:
    result.status failed → state failed, error
    result.classification.needs_classification → state needs_input, needs={"type":"chart_type","candidates":[...],"preview_url":f"/api/jobs/{id}/preview.jpg"}
    result.input.raw_meta.needs_input → state needs_input, needs={"type":"raw_meta","candidates":[...]}
    그 외 → 서술: auto 면 llm.narrative(B6) 시도, 실패·없음이면 템플릿. write_report. state done(ok) 또는 partial.
  예외는 state failed, error 메시지. 절대 서버를 죽이지 않는다.
- resume(job_id, params): state 가 needs_input 이 아니면 E_JOB_STATE. params 병합 후 run.
- set_manual_narrative(job_id, narrative: Narrative): write_report(source="manual"), job.result 갱신.
- get(job_id): 없으면 E_JOB_NOT_FOUND. summary_list(): [{job_id, filename, state, chart_type, created_at}].
- 큐: asyncio.Lock 또는 threading.Lock 으로 run 을 직렬화한다.

web/api.py — APIRouter(prefix="/api")
  GET /health → {ok: true, llm: {configured: bool, mode: "auto"|"manual", model, probe: dict|null}}
  GET /catalog → registry.catalog_dict()
  POST /jobs (multipart: file, chart_type?, project_id?, width?, height?, bit_depth?, bayer?, use_llm? 기본 true)
    → 검증: chart_type/project_id/bayer 가 허용값 아니면 400 {"error":{"code":"E_UNKNOWN_CHART_TYPE"|...,"message":...,"allowed":[...]}}
    → 202 {"job_id"}; BackgroundTasks 로 manager.run
  GET /jobs/{id} → Job(결과 포함). result.basic 에서 "basic.uniformity.grid" 는 유지, 그 외 큰 값 없음.
  POST /jobs/{id}/resume (JSON) → 검증 후 202 {"job_id"}
  GET /jobs/{id}/preview.jpg → FileResponse
  GET /jobs/{id}/report.md → PlainTextResponse(media_type="text/markdown"), 없으면 404
  GET /jobs/{id}/interpret-prompt → PlainTextResponse(build_interpret_prompt)
  POST /jobs/{id}/narrative (JSON {summary,interpretation,recommendation} 또는 {markdown}) → parse_narrative → set_manual_narrative → {report_path, narrative_source:"manual"}
  GET /jobs → summary_list()  [P1 이지만 10줄이면 지금 넣어라]
  ChartAgentError 는 전역 예외 핸들러로 {"error":{code,message}} 와 400/404 로 변환.

web/app.py — create_app(registry_path=None) -> FastAPI. lifespan 에서 Registry.load, llm 은 환경 변수 GLM_BASE_URL 이 있으면 B6 의 클라이언트
  (지금은 None), JobManager 생성해 app.state 에 보관. router 포함. static/ 을 "/" 에 StaticFiles(html=True) 로 마운트(API 라우터를 먼저 등록).
  CORS 불필요(같은 origin). faulthandler.enable().

cli.py — serve [--host 127.0.0.1] [--port 8000] [--registry] → uvicorn.run(create_app(...), host, port).

tests/test_api.py — fastapi.testclient.TestClient 로:
  (a) sfr_test.bmp 업로드 → 폴링(최대 30회, sleep 0.2) → done/partial → report.md 200, 판정 표 포함.
  (b) photo_test.jpg → needs_input(type chart_type, candidates 비어 있지 않음, preview 200) → resume {chart_type:"flat_field"} → done/partial.
  (c) GLM_BASE_URL 없이 → health.llm.mode == "manual" → interpret-prompt 200 → narrative POST(markdown "## 요약...## 해석...## 권고...") → narrative_source manual, report 에 반영.
  (d) chart_type="nope" → 400, allowed 포함. (e) .txt 업로드 → 400 E_UNSUPPORTED_FORMAT.
  BackgroundTasks 가 TestClient 에서 동기 실행되는 점을 이용하거나, manager.run 을 직접 호출해도 된다.

완료 조건: pytest 통과. chart-agent serve 후 브라우저에서 http://127.0.0.1:8000/api/health 가 보인다.
```

---

## 06 작업 B5: 프런트엔드 (4:30~6:00)

```text
[작업 B5: 정적 프런트엔드] P0

파일: static/index.html, static/app.js, static/style.css. 외부 라이브러리·CDN 금지. 최신 크롬 기준. 한국어 UI.
화면은 세로로 쌓인 카드 4개다. 필요 없는 카드는 hidden 클래스로 숨긴다.

1. 업로드 카드 (#card-upload)
- 파일 input(여러 파일 허용은 P1, 지금은 1개) + 드롭 영역. 선택한 파일명·크기 표시.
- 차트 유형 select(기본 "자동 판별", 옵션은 GET /api/catalog 의 chart_types), 과제 select(기본 "자동").
- raw 필드 4개(width, height, bit_depth, bayer select): 파일 확장자가 .raw 일 때만 표시.
- "LLM 사용" 체크(기본 on). GET /api/health 의 llm.mode 를 배지로 표시(auto/manual).
- "분석" 버튼 → POST /api/jobs(multipart) → job_id 저장 → 진행 카드 표시.

2. 진행 카드 (#card-progress)
- 1초 setInterval 로 GET /api/jobs/{id}. state 배지(queued/running/needs_input/done/partial/failed), steps 를 목록으로(최근 것이 아래).
- state 가 needs_input 이면 확인 카드, done/partial 이면 결과 카드, failed 면 오류 메시지. 폴링 중단 조건을 정확히.

3. 확인 카드 (#card-confirm)
- needs.type == "chart_type": 프리뷰 img(needs.preview_url) + 후보 라디오 목록(candidates 순서, confidence 와 reason 표시, 카탈로그 전체 포함) + "선택 후 재개" 버튼
  → POST /api/jobs/{id}/resume {chart_type}.
- needs.type == "raw_meta": width/height/bit_depth/bayer 폼(첫 후보로 미리 채움, 후보 목록을 클릭하면 폼에 채움) + "재개" → resume.

4. 결과 카드 (#card-result)
- 상단: 파일명, 차트 유형(판별 방법·신뢰도), 과제, 상태 배지, 서술 출처 배지(llm/template/manual).
- 프리뷰 img. 판정 표: 합격 초록, 불합격 빨강, 참고 회색(passed null). 기본 평가 표(항목별 키·값), 과제 결과 표 또는 dll_status·error.
- 보고서: GET /api/jobs/{id}/report.md → 간단 렌더러(제목 #/##, 표 |, 목록 -, 인용 >, 그 외 문단)로 HTML 변환, 실패 시 <pre>. "report.md 다운로드" 링크.
- 수동 서술 영역(서술 출처가 template 일 때 펼침): "해석 프롬프트 복사" 버튼(GET interpret-prompt → clipboard), textarea, "서술 적용" 버튼
  → POST /api/jobs/{id}/narrative {markdown} → 보고서 다시 로드.
- "새 분석" 버튼 → 업로드 카드로.

app.js 규칙: fetch 래퍼 하나(api(path, opts))가 {error} 응답을 throw 로 바꾼다. 전역 상태는 state 객체 하나. 함수는 카드별로 render* 로 나눈다.
style.css: 폭 960px 중앙, 카드 그림자, 배지 색(초록/빨강/회색/파랑), 표 줄무늬. 다크 모드 불필요.

완료 조건: chart-agent serve 후 브라우저에서 sfr 샘플 업로드 → 결과 카드 → 보고서. unknown 샘플 → 확인 카드 → 선택 → 결과. manual 모드에서 수동 서술 적용.
```

---

## 07 작업 B6: LLM 클라이언트·판별·서술 (6:00~7:15, API 있으면 B5 앞으로)

```text
[작업 B6: LLM 클라이언트] P0.5

파일: src/chart_agent/llm/__init__.py, llm/client.py, llm/mock.py, llm/probe.py, llm/prompts.py, prompts/classify.md, prompts/narrative.md,
tests/test_llm.py. pipeline.py(llm.classify 연결), web/jobs.py(llm.narrative 연결), web/app.py(클라이언트 생성·probe) 는 전체 다시 출력.

llm/client.py
  class LlmClient(Protocol):
      def classify(self, preview_jpeg: bytes, filename: str, catalog: dict) -> ChartClassification: ...
      def narrative(self, tables_md: str, context: dict) -> Narrative: ...
      def chat(self, messages: list[dict], tools: list[dict]) -> dict: ...     # B7
      def probe(self) -> dict: ...
  class OpenAICompatibleClient(LlmClient):
      __init__(settings: dict from registry.llm_settings(), base_url, api_key): openai.OpenAI(base_url, api_key, timeout).
      _complete_json(system, user_text, images: list[bytes], schema_model, section) -> BaseModel:
         이미지는 {"type":"image_url","image_url":{"url":"data:image/jpeg;base64,..."}} 로 user content 에 첨부.
         구조화 출력 3단계: self.json_mode 가 "schema" 면 response_format json_schema(strict, schema=model_json_schema()), "object" 면 json_object,
         "prompt" 면 user 끝에 "JSON 만 출력하라. 스키마: ..." 추가 후 응답에서 첫 {...} 블록 추출. pydantic 검증 실패 시 오류 메시지를 붙여 1회 재요청.
         temperature/max_tokens 는 settings[section]. reasoning_effort 는 extra_body 로(settings 에 값 있을 때만). 네트워크·5xx·429 는 retries 만큼 지수 백오프,
         4xx 는 즉시 ChartAgentError("E_LLM_UNAVAILABLE"). 파싱 실패는 E_LLM_PARSE.
      classify: system = prompts/classify.md 의 system 절(카탈로그 치환), user = 파일명·해상도 텍스트 + 프리뷰 1장. 응답 chart_type 이 카탈로그에 없으면 "unknown".
         method "llm", needs_classification = confidence < threshold.
      narrative: system = prompts/narrative.md system 절, user = context(과제 설명, 차트 유형, 판별 방법) + tables_md. Narrative 반환.
      probe: scripts/probe_llm.py 와 같은 P1~P7 을 실행해 dict 반환, json_mode 를 결과로 결정("schema" > "object" > "prompt").
  create_client_from_env(registry) -> LlmClient | None: GLM_BASE_URL 없으면 None.

llm/mock.py — MockLlmClient(classify_result=None, narrative_result=None, fail=False). 테스트·개발용 고정 응답. fail=True 면 E_LLM_UNAVAILABLE.
llm/prompts.py — prompts/*.md 를 읽어 "## system" "## user" 절로 나누고 {catalog} 등 치환. 파일 해시를 반환(result.json 기록용).

prompts/classify.md 와 prompts/narrative.md 는 내가 별도로 주는 텍스트(런타임 프롬프트 20, 21)를 그대로 넣는다. 지금은 자리만 만들어라.

pipeline.py: unknown 이고 llm 이 있으면 llm.classify 호출, confidence >= threshold 면 채택(needs_classification False), 아니면 unknown 유지하되 candidates 를 LLM 것으로 교체.
  LLM 예외는 errors 에 E_LLM_* 로 넣고 규칙 결과 유지.
web/jobs.py: auto 모드에서 render_tables 로 tables_md 를 만들어 llm.narrative 호출 → write_report(source="llm"). 실패 시 템플릿 + errors.
web/app.py: create_client_from_env 로 클라이언트 생성, 시작 시 probe 1회(실패해도 서버는 뜬다), /api/health 에 노출.

tests/test_llm.py — MockLlmClient 로: unknown 이미지가 classify_result(sfr, 0.9) 로 채택되는지, fail=True 면 템플릿 서술로 done/partial 되는지,
  _complete_json 의 "prompt" 모드 JSON 블록 추출 단위 테스트(가짜 응답 문자열).

완료 조건: pytest 통과. GLM_BASE_URL 설정 후 serve → health.llm.mode auto → sfr 샘플 업로드 → 서술 출처 llm.
```

---

## 08 작업 B7: raw 추정·채팅 에이전트 (7:15~8:30)

```text
[작업 B7: raw 메타 추정과 채팅 에이전트] P1

A. src/chart_agent/loaders/raw_estimate.py — estimate_raw_meta(path, known_resolutions, default_bayer) -> RawMeta
  1. S=파일 크기. b∈{2,1} 에 대해 N=S/b 가 정수인 경우만.
  2. w*h==N 인 (w,h) 를 known_resolutions + 기본 표(640x480,1280x720,1280x960,1920x1080,2592x1944,3264x2448,4000x3000,4032x3024,4608x3456,8000x6000) 에서.
     없으면 w%8==0 이고 종횡비 4:3,3:2,16:9,1:1(오차 2%) 약수 쌍.
  3. 각 후보 (h,w) reshape, 32행 간격 샘플 행 r 과 r+2 평균 절대 차이가 점수. 최저 채택, confidence=clamp(1-best/second,0,1), 하나면 0.9.
  4. 비트: 최댓값이 들어가는 최소 비트 {8,10,12,14,16}. b==2 이고 max<=255 면 8·16 둘 다 후보.
  5. 베이어: 4위상 평균 중 가장 가까운 두 평면이 G. (0,1),(1,0) 이면 RGGB|BGGR, (0,0),(1,1) 이면 GRBG|GBRG. R/B 는 default_bayer 우선, 아니면 둘 다 후보.
  6. estimated=True, confidence<0.8 또는 후보 2개 이상이면 needs_input=True, candidates 최대 5개.
  resolve_raw_meta 가 모든 소스가 없을 때 이 함수를 호출하도록 raw.py 수정. pipeline 은 needs_input 이면 errors 에 E_RAW_ESTIMATE_AMBIGUOUS, partial.
  프런트엔드 확인 카드의 raw_meta 폼이 candidates 로 채워지는지 확인.
  tests: 160x120 RGGB 10비트 합성 raw(사이드카 없음) → width/height/bit_depth 일치, bayer ∈ {RGGB,BGGR}.

B. src/chart_agent/web/chat.py + prompts/chat.md + POST /api/jobs/{id}/chat + 프런트엔드 결과 카드에 채팅 영역
  - 도구 3개(파이썬 함수 + OpenAI tools 스키마):
    get_result() → 현재 작업의 result.json 요약(classification, project.results, judgement, basic.sharpness/clipping/noise).
    run_analysis(analysis_id: enum=registry.all_analysis_ids(), roi: "center"|"full"|{x,y,w,h}) → dll.run_single_analysis 결과.
    get_catalog() → catalog_dict.
  - 루프: messages=[system(prompts/chat.md + 작업 요약), user], 최대 chat.max_steps. 매 턴 llm.chat(messages, tools) → tool_calls 가 있으면 순서대로 실행해
    role "tool" 메시지로 추가, 없으면 answer 반환. 도구 오류는 {"error":..., "allowed":[...]} 를 도구 결과로 돌려준다(예외 아님).
  - 응답 {"answer", "tool_calls":[{"name","args","result_summary"}]}. OpenAICompatibleClient.chat 은 tools/tool_choice="auto" 로 1회 호출해
    {"content","tool_calls":[{"id","name","arguments"}]} 를 돌려준다. MockLlmClient.chat 은 스크립트된 턴 목록을 순서대로 반환.
  - 프런트엔드: 결과 카드 아래 입력창 + 전송, 대화 목록, 도구 호출 내역을 접이식으로 표시. llm.mode 가 auto 가 아니면 숨김.
  - tests: MockLlmClient 로 run_analysis 1회 → 답변 시나리오, 없는 analysis_id 요청 → allowed 포함 도구 결과.

완료 조건: pytest 통과. 사이드카 없는 raw 로 확인 카드가 뜨고 폼이 채워진다. auto 모드에서 채팅 질문에 도구 호출 내역과 답이 보인다.
```

---

## 09 작업 B8: 다중 파일·오류 경로·로그·격리 (8:30~9:30)

```text
[작업 B8: 일괄 처리, 오류 경로, 로그, 격리 옵션] P1

1. 다중 파일: 업로드 카드에서 여러 파일 선택 → 파일마다 POST /api/jobs → 작업 목록 카드(#card-jobs: GET /api/jobs 를 2초마다, 행 클릭으로 결과 카드 전환).
2. 서버 재시작 복구: JobManager 시작 시 out/*/job.json 을 읽어 jobs 복원(running 이던 것은 failed 로 표시).
3. 로그: 각 작업의 steps 에 elapsed_ms 를 채우고, LLM 호출은 steps 에 {"stage":"llm.classify"|"llm.narrative","elapsed_ms","tokens"} 기록. 서버는 out/server.log 로 logging.
4. pipeline.isolate == "subprocess" 면 JobManager.run 이 pipeline.run 대신 <<PYTHON_EXE>> -m chart_agent.cli analyze ... 를 subprocess.run(timeout=120) 으로 실행하고
   result.json 을 읽는다. 비정상 종료는 failed + stderr 앞 500자.
5. 오류 경로 테스트: 크기 불일치 raw → failed + 기대 크기 메시지, mock fail → partial + dll_status failed, 없는 analysis_id 채팅 도구 → allowed,
   200MB 초과 → 400 E_UPLOAD_TOO_LARGE(파일 크기 검사는 Content-Length 또는 읽은 바이트 수).
6. 보고서 서술 숫자 검증(선택): write_report 에서 서술 속 숫자를 추출해 judgement/basic 값 집합에 없으면 errors 에 경고 기록(거부는 하지 않음).

변경 파일 전체 다시 출력. 완료 조건: pytest 전체 통과, 샘플 4개 동시 업로드 → 작업 목록 → 각 결과.
```

---

## 10 작업 B9: README·시연 준비 (9:30~10:30)

```text
[작업 B9: README 와 시연 준비] P1

1. README.md: 설치(setup.ps1), 환경 변수(GLM_BASE_URL, GLM_API_KEY, CHART_AGENT_REGISTRY), 실행(chart-agent serve, 브라우저 http://127.0.0.1:8000),
   레지스트리 설명과 과제 추가(YAML 블록 + 어댑터 1파일 + kind 변경), CLI 사용법, LLM 모드(auto/manual/off)와 수동 서술 절차,
   문제 해결(WinError 193/126, 포트 충돌, LLM 401/404, json_schema 미지원, raw 추정 틀림), 보안 고지(로컬 전용·인증 없음).
2. demo/ 구성 안내: sfr_01.bmp, sfr_02.jpg, flat_01.bmp, unknown_01.jpg, sensor_01.raw(사이드카 없음) 와 기대 결과 표.
3. scripts/rehearsal_checklist.md: 서버 기동 → health 확인 → 시연 대사 6개 → 각 단계 소요 시간 기록란 → 실패 시 대체 동작.
4. 발표용 구조도(텍스트)와 "Agent 다운 요소" 5줄(자동 판별·확인 요청·DLL 계산·LLM 해석·채팅 도구 호출).

완료 조건: README 만 읽고 다른 사람이 설치·실행·시연을 재현할 수 있다.
```

---

## 20 런타임: 판별 프롬프트

`prompts/classify.md`에 그대로 넣는다. `{catalog}`는 레지스트리에서 "- id: name — description" 줄로 치환한다.

```text
## system
당신은 카메라 화질 평가용 테스트 차트 분류기다. 입력 이미지가 아래 카탈로그 중 어떤 차트인지 판별한다.

카탈로그:
{catalog}
- unknown: 위 어느 것에도 해당하지 않거나 확신이 없음

규칙:
- chart_type 은 카탈로그의 id 또는 "unknown" 만 쓴다. 다른 문자열은 금지.
- confidence 는 0.0~1.0. 두 유형이 비슷하게 보이면 0.6 이하로 두고 candidates 에 둘 다 넣는다.
- candidates 는 가능성 순으로 최대 3개. 각 reason 은 눈에 보이는 근거 한 문장(예: "기울어진 검은 사각형이 격자로 배치됨").
- 사진·풍경·문서처럼 차트가 아니면 "unknown", confidence 0.9 이상.
- 수치 측정이나 품질 평가는 하지 않는다. 유형 판별만 한다.
- 출력은 JSON 하나. 설명 문장 금지.

출력 스키마:
{"chart_type": "string", "confidence": 0.0, "candidates": [{"chart_type": "string", "confidence": 0.0, "reason": "string"}]}

## user
파일명: {filename}
해상도: {width}x{height}, 포맷: {format}
아래 이미지를 보고 차트 유형을 판별하라.
```

## 21 런타임: 서술 프롬프트

`prompts/narrative.md`에 그대로 넣는다.

```text
## system
당신은 카메라 화질 평가 보고서 작성자다. 주어진 수치 표와 판정 표만 근거로 한국어 서술 3개를 쓴다.

규칙:
- 표에 있는 값만 인용한다. 단위와 소수점을 그대로 옮기고 새 수치를 만들거나 환산·반올림하지 않는다.
- 판정 결과(합격/불합격/참고)를 바꾸지 않는다.
- summary: 3문장 이내. 차트 유형, 전체 상태, 불합격 건수.
- interpretation: 불합격과 참고 항목 중심. 각 항목의 값과 기준을 "mtf50=0.21 (기준 >= 0.30)" 형식으로 인용. 불합격이 없으면 가장 여유가 적은 항목을 언급.
- recommendation: 재촬영·초점·노출·조명·차트 정렬 등 다음 행동 2~3개. 각 권고에 근거 항목을 붙인다. DLL 실패나 unknown 차트가 있으면 그 해결을 첫 권고로.
- 출력은 JSON 하나. 설명 문장 금지.

출력 스키마:
{"summary": "string", "interpretation": "string", "recommendation": "string"}

## user
과제: {project_name} — {project_description}
차트 유형: {chart_type} (판별 {method}, 신뢰도 {confidence})  상태: {status}
문제: {errors}

{tables_md}
```

## 22 런타임: 채팅 에이전트 프롬프트와 도구 스키마

`prompts/chat.md`와 `web/chat.py`의 도구 스키마.

```text
## system
당신은 방금 분석된 차트 이미지에 대해 사용자의 질문에 답하는 화질 평가 보조자다. 아래 도구만 사용한다.

현재 작업 요약:
{job_summary}

규칙:
- 수치는 도구 결과에서만 인용한다. 기억이나 추정으로 숫자를 말하지 않는다.
- 질문에 답하려면 먼저 get_result 를 호출해 현재 값을 확인한다. 다른 영역이나 항목의 값이 필요하면 run_analysis 를 호출한다.
- 도구는 한 번에 하나만 호출하고 결과를 본 뒤 다음을 결정한다. 최대 5회.
- 등록되지 않은 분석을 요구받으면 get_catalog 의 목록을 보여주고 "등록되지 않은 분석"이라고 답한다.
- 도구가 error 와 allowed 를 돌려주면 allowed 안에서 고쳐 다시 호출한다.
- 답변은 한국어 5줄 이내. 인용한 수치와 그 출처(항목 이름)를 명시한다.
```

```json
[
  {"type": "function", "function": {"name": "get_result",
    "description": "현재 작업의 분석 결과 요약(차트 유형, 과제 결과, 판정 표, 선명도·클리핑·노이즈)을 반환한다. 질문에 답하기 전에 먼저 호출한다.",
    "parameters": {"type": "object", "properties": {}}}},
  {"type": "function", "function": {"name": "run_analysis",
    "description": "등록된 분석 항목 하나를 지정 ROI로 다시 실행하고 그 결과만 반환한다. 사용자가 다른 영역이나 항목의 값을 물을 때 쓴다.",
    "parameters": {"type": "object",
      "properties": {"analysis_id": {"type": "string", "enum": ["<<레지스트리 all_analysis_ids()>>"]},
                     "roi": {"description": "center | full | {x,y,w,h} (0~1 비율)", "anyOf": [{"type": "string"}, {"type": "object"}]}},
      "required": ["analysis_id"]}}},
  {"type": "function", "function": {"name": "get_catalog",
    "description": "등록된 차트 유형·과제·분석 항목·임계값 목록을 반환한다.",
    "parameters": {"type": "object", "properties": {}}}}
]
```

## 23 수동 대체 경로: 해석 요청 템플릿

B3의 `prompts/interpret_template.md`와 같다. 백엔드가 `GET /api/jobs/{id}/interpret-prompt`로 표를 채워 돌려주고, 사용자가 사내 채팅 프로그램에 붙여넣는다. 답을 결과 카드의 수동 서술 영역에 붙여넣고 "서술 적용"을 누르면 `## 요약`·`## 해석`·`## 권고` 절을 파싱해 보고서에 반영한다. 붙여넣은 답에 JSON이 섞여 있어도 파싱된다.

## 24 런타임: 환경 변수와 실행

```text
# .env.example (실제 값은 환경 변수로만. 저장소에 올리지 않는다)
GLM_BASE_URL=<<LLM_BASE_URL>>
GLM_API_KEY=<여기에 키>
CHART_AGENT_REGISTRY=<<WORK_DIR>>\config\registry.yaml

# 실행 (PowerShell)
$env:GLM_BASE_URL="<<LLM_BASE_URL>>"; $env:GLM_API_KEY="<키>"
<<PYTHON_EXE>> -m chart_agent.cli serve --port 8000
# 브라우저 http://127.0.0.1:8000  /  상태 http://127.0.0.1:8000/api/health
# manual 모드 시연: GLM_BASE_URL 을 비우고 serve
```

---

## 30 시연 대사

| 순서 | 동작 | 기대 결과 |
|---|---|---|
| 1 | `/api/health` 열기 | `llm.mode` auto(또는 manual) 배지 |
| 2 | `sfr_01.bmp` 업로드 | 진행 카드 → 결과 카드: 판별 rule, 판정 표, 서술 출처 llm |
| 3 | `unknown_01.jpg` 업로드 | 확인 카드: 프리뷰 + 후보(LLM reason 포함) → `flat_field` 선택 → 결과 |
| 4 | `sensor_01.raw` 업로드(사이드카 없음) | 확인 카드 raw 폼이 추정값으로 채워짐 → 재개 → 결과(베이어 통계 포함) |
| 5 | 결과 카드 채팅: "중앙 MTF50이 왜 낮아? 전체 영역은?" | 도구 호출 내역(get_result, run_analysis full) + 수치 인용 답변 |
| 6 | 레지스트리에 `flat_field` 임계값 추가 후 `flat_01.bmp` 업로드 | 코드 변경 없이 판정 반영 |
| 7 | (manual 모드) `sfr_02.jpg` 업로드 → 해석 프롬프트 복사 → 사내 채팅에 붙여넣기 → 답 붙여넣기 → 서술 적용 | 서술 출처 manual 보고서 |

리허설에서 2번의 업로드→결과 시간을 재고 2분을 넘기면 `reasoning_effort`를 낮추거나 서술 `max_tokens`를 줄인다.

## 31 문제 해결 프롬프트 (사내 LLM에 붙여넣기)

| 증상 | 프롬프트 |
|---|---|
| 업로드 후 진행 카드가 멈춤 | "GET /api/jobs/{id} 가 running 에서 멈춘다. JobManager.run 이 예외를 삼키고 state 를 failed 로 바꾸는지, BackgroundTasks 가 실행되는지 확인하는 로그를 추가하고, 60초 넘게 running 이면 프런트가 '서버 로그 확인' 안내를 띄우게 수정하라." |
| LLM 401/404 | "OpenAICompatibleClient 가 401/404 를 받으면 E_LLM_UNAVAILABLE 로 즉시 실패하고 JobManager 는 manual 모드로 작업을 이어 가도록 보장하라. /api/health 에 마지막 LLM 오류 메시지를 노출하라." |
| json_schema 미지원 | "probe 에서 P3 실패·P4 성공이면 json_mode 를 'object' 로, 둘 다 실패면 'prompt' 로 자동 설정되는지 확인하고 테스트를 추가하라. 'prompt' 모드의 JSON 추출이 코드 블록(```json) 안에 있어도 동작하게 하라." |
| 서술에 없는 숫자 | 21 프롬프트 첫 규칙을 "표의 값을 문자 그대로 복사한다. 복사할 수 없는 수치는 쓰지 않는다"로 바꾸고, B8-6 숫자 검증을 거부(재요청 1회)로 올린다: "write_report 전 서술 숫자 검증 실패 시 오류 메시지를 붙여 llm.narrative 를 1회 재호출하고 그래도 실패면 템플릿으로." |
| WinError 193 | "DLL 이 32비트다. 레지스트리 dll.python_exe 에 32비트 Python 경로를 받아 어댑터 호출을 그 인터프리터의 subprocess 로 실행하는 dll/worker.py 를 작성하라. 어댑터 인터페이스는 그대로." |
| WinError 126 | "os.add_dll_directory 로 DLL 폴더를 검색 경로에 추가하고, PE import table 을 읽어 누락 DLL 이름을 check_dll.py 가 출력하게 하라." |
| DLL 크래시로 서버 종료 | "registry pipeline.isolate 를 subprocess 로 바꾸면 작업이 CLI 서브프로세스로 돌도록 B8-4 를 구현하고, 크래시 시 failed + stderr 로 기록되는지 테스트하라." |
| raw 추정 틀림 | "raw_estimate 점수에 열 방향(c, c+2) 차이를 더하고 known_resolutions 후보에 가중치 0.5 를 곱해 우선하라. 실제 샘플 크기 {S} 바이트 케이스 테스트 추가." |
| 결과 카드 느림 | "GET /api/jobs/{id} 응답에서 basic.uniformity.grid 와 *_std 를 제거하고(파일은 유지), 폴링 간격을 running 일 때 1초, needs_input 이후 중단으로 고쳐라." |
| 포트 충돌 | `chart-agent serve --port 8001` 로 실행하고 브라우저 주소를 바꾼다. |

## 32 자체 점검 프롬프트 (코드 완성 후)

```text
[자체 점검]
아래 인수 조건마다 현재 코드가 만족하는지 "조건 | 근거 파일·함수 | 통과/미통과/미확인 | 미통과면 수정 제안" 표로 답하라. 추측하지 말고 코드에서 근거를 찾아라.
AC-01 bmp/jpg 로드 시 폭·높이·채널이 원본과 일치
AC-02 raw + 메타 로드 성공, 크기 불일치면 E_RAW_SIZE_MISMATCH
AC-03 파일명 규칙으로 sfr 샘플은 sfr_iso12233, 규칙 없는 파일은 unknown 또는 LLM 후보 + 사용자 선택
AC-04 proj_sfr 분석 시 results 에 proj_sfr.center.mtf50, dll_status ok
AC-05 DLL 예외 시 dll_status failed, 기본 평가 유지, 서버 생존
AC-06 CLI 단독 실행으로 result.json 과 템플릿 report.md 생성
AC-07 report.md 판정 표 수치 == result.json judgement 값
AC-08 레지스트리에 유형·과제 추가만으로 동작
AC-09 없는 chart_type/analysis_id 로 API·도구 호출 시 error 와 allowed
AC-10 같은 입력 2회 실행 시 수치 동일
AC-11 브라우저에서 업로드 1회로 결과 카드와 report.md 다운로드 (TestClient 업로드→폴링→done→report 테스트 포함)
AC-12 사이드카 없는 raw 추정 폭·높이·비트 일치
AC-13 GLM_BASE_URL 없이 interpret-prompt → narrative POST → narrative_source manual
AC-14 unknown 업로드 → needs_input → resume → done
AC-15 auto 모드 채팅이 도구 결과 수치만 인용하고 tool_calls 를 응답에 포함
미통과 항목이 있으면 수정 코드를 파일 전체로 출력하라.
```

## 33 컨텍스트 복구 프롬프트 (새 대화로 옮길 때)

`00 브리프`를 먼저 붙인 뒤 이어서:

```text
[진행 상황]
완료: {완료한 작업 번호와 파일 목록, 예: B0~B4, models.py registry.py loaders/* basic.py pipeline.py cli.py dll/* judge.py report.py web/*, pytest 전부 통과}
현재 파일 중 당신이 알아야 할 것: {주요 파일 전체 붙여넣기. 최소 models.py, registry.py, pipeline.py, web/jobs.py, web/api.py}
지금 할 작업: {다음 작업 번호}
위 파일과 호환되게 작성하라. 기존 함수 시그니처와 API 경로를 바꾸지 않는다.
```

## 변경 이력

| 일자 | 변경 | 근거 | 상태 |
|---|---|---|---|
| 2026-10-13 | 최초 작성. MCP 프롬프트 팩을 웹 앱 구조로 재작성 | 사용자 결정(웹 개발로 목표 변경) | `awaiting-approval` |
