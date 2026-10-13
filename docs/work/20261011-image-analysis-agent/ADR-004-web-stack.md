# ADR-004: 웹 기술 스택 선택

> 문서 유형: `adr`
> 작업 ID: `20261011-image-analysis-agent`
> 상태: `proposed`
> 기준선: `draft`
> 작성일: 2026-10-13
> 최종 갱신: 2026-10-13
> 관련 문서: [REQ-image-analysis-agent: 요구사항](../../requirements.md), [DESIGN-image-analysis-agent: 설계](../../design.md)

## 요약

- 목적: 12시간 해커톤에서 사내 LLM이 생성하는 코드로 완성할 수 있는 웹 스택을 정한다.
- 현재 결론 또는 상태: 백엔드는 FastAPI + uvicorn, 프런트엔드는 빌드 도구·CDN 없는 정적 HTML/CSS/JS, 상태 저장은 파일 시스템. 승인 대기.
- 다음 행동: 설계 승인과 함께 `approved`.

## 문서 연결

| 방향 | 관계 | 대상 문서 | 대상 항목 | 비고 |
|---|---|---|---|---|
| input | baseline | [REQ-image-analysis-agent: 요구사항](../../requirements.md) | FR-07, NFR-01, NFR-08, C-01, C-03, C-06, RISK-09 | 결정이 적용되는 요구사항 |
| output | decision | [DESIGN-image-analysis-agent: 설계](../../design.md) | DES-10, DES-11, 8절 | 이 결정을 반영한 설계 |
| output | related | [결정 등록부](../../decisions.md) | ADR-004 | 등록 |

## 배경

코어가 Python이고 데이터 모델이 pydantic이다. 해커톤 장소는 오프라인일 수 있고(A-09) 코드는 사내 LLM이 프롬프트로 생성한다. 프런트엔드에 쓸 수 있는 시간은 1시간 30분 안팎이다(RISK-09).

## 고려한 대안

| 안 | 백엔드 | 프런트엔드 | 장점 | 단점 |
|---|---|---|---|---|
| A | FastAPI + uvicorn | 정적 HTML + 바닐라 JS | pydantic 모델 재사용, 업로드·백그라운드 작업 내장, 의존성 적음, 오프라인 동작, LLM이 생성하기 쉬움 | 화면 꾸밈에 한계 |
| B | FastAPI | React/Vue + 빌드 도구 | 풍부한 UI | 빌드 환경·npm 의존, 오프라인 위험, 시간 초과 |
| C | Flask | Jinja 서버 렌더링 | 단순 | 폴링·업로드 처리 코드가 늘고 비동기 작업 관리가 번거로움 |
| D | Streamlit/Gradio | 내장 | 가장 빠름 | 작업 상태 기계·확인 요청·REST API 노출이 어색하고 Agent 구조 설명이 어려움 |

## 결정

A를 채택한다.

- 백엔드: FastAPI, uvicorn, python-multipart. 작업은 `BackgroundTasks`로 순차 실행. 상태는 인메모리 + `out/<job_id>/job.json`.
- 프런트엔드: `static/index.html`, `app.js`, `style.css` 3파일. 외부 라이브러리·CDN 없음. 보고서 Markdown은 60줄 이내 간단 변환 함수로 렌더링하고 실패 시 `<pre>`.
- 화면은 카드 4개(업로드·진행·확인 요청·결과)로 제한한다. 꾸밈은 리허설 이후에만 한다.
- 테스트는 `fastapi.testclient.TestClient`(httpx)로 업로드→폴링→결과 흐름을 자동화한다.

## 결과

- 장점: 코어·모델을 그대로 노출하고 의존성이 작아 오프라인에서도 설치가 쉽다. 사내 LLM이 생성하는 코드 양이 가장 적다.
- 감수할 단점: UI 완성도는 낮다. 다중 사용자·동시 처리는 범위 밖이다.
- 영향 범위: DES-10, DES-11, 설계 8절, 9.2 타임라인 3:30~6:00.

## 변경 이력

| 일자 | 변경 | 근거 | 상태 |
|---|---|---|---|
| 2026-10-13 | 최초 작성 | 목표 변경(웹 개발) | `proposed` |
