# 테스트 지원 포털(Test Support Portal) 명세서

> **문서 성격**: 이미 구현된 시스템을 합의·인수인계용으로 정리한 **as-built(현행 구현 기준) 명세**다.

> 모든 내용은 현재 코드(`backend/main.py`, `models.py`, `schemas.py`, `frontend/src`, `run.sh`) 기준이며,

> 코드에서 단정할 수 없는 항목은 [§11 미해결 질문](#11-미해결-질문)에 모았다.

---

## 1. 배경 / 문제

QA 담당자는 운영(PROD)·개발(DEV) 등 여러 환경에 흩어진 웹 시스템을 점검해야 한다. 기존 방식은

사람이 사이트 구조를 일일이 파악하고, 페이지마다 수동 체크리스트를 돌리며, 운영과 개발이 동일하게

배포됐는지 눈으로 비교하는 식이라 느리고 누락이 많다.

이 포털은 그 흐름을 자동화한다.

1. **시스템 + 환경 URL 등록** →
2. 백그라운드가 사이트를 **크롤**해 구조를 **트리**로 시각화 →
3. 페이지별로 **자동(건강성)/AI 테스트**를 돌려 **URL별 결과 저장소**를 채우고 →
4. **운영↔개발을 AI로 비교**해 증적 리포트를 만든다.

핵심 데이터는 `(environment_id, url)` 키로 보존되는 **TestResult(URL별 결과 저장소)** 이며,

사이트맵/결과/리포트 탭이 모두 이 저장소를 읽는다.

**사용자(Who).** 단일 QA 담당자가 아니라, 사내 다수 시스템을 **시스템별로 나눠 맡은 담당자 여러 명**이다. 점검 대상 시스템 수만큼 담당자가 있으므로 혜택 범위가 조직 전반으로 넓다. **포인트는, 이들이 손으로 하던 반복 점검·운영↔개발 비교를 AI가 대신 수행**한다는 것 — 사람의 수작업을 자동화해 누락을 줄이고 시간을 돌려준다.

---

## 2. 목표 / 범위 밖

### 목표 (Goals)

- 시스템·환경을 등록하면 환경별 크롤이 **동시에** 비동기로 돌아 사이트 트리를 만든다.
- 크롤된 페이지에 대해 **API 키 없이도** 건강성 점검(HTTP 상태·깨진 링크/이미지·alt 누락 등)을 수행한다.
- 키가 있으면 상위 페이지를 **Claude로 AI 평가**하고, **운영↔개발 AI 비교** 증적을 생성한다.
- 재크롤/재실행해도 URL별 결과가 **보존**되며, 자동 엔진과 사람의 수동 입력이 같은 저장소에 공존한다.
- 진행 상태(pending/running/done/error)와 테스트 진행률을 폴링으로 노출한다.

### 범위 밖 (Non-goals)

- **사람 체크리스트 방식이 아니다.** 테스트는 AI/자동 엔진이 URL별 결과 저장소를 채우는 방식이다.
- **인증/권한/멀티테넌시 없음.** 로그인·사용자 모델·접근 제어 코드가 없다(외부 오픈 바인딩 주의 → §9).
- **외부 CI/이슈 트래커 연동 없음.** 결과는 SQLite 단일 파일에만 저장된다.
- **Alembic 등 정식 마이그레이션 도구를 쓰지 않는다.** `main.py::_migrate()`의 수작업 ALTER가 전부다.

### 성공 기준 (7/15 Definition of Done)

**하나의 골든 패스가 끊김 없이 도는 것**을 성공으로 정의한다:

회사 소개 사이트의 **운영·개발 환경을 둘 다 등록** → 환경별 크롤 `done` → URL별 자동/AI 테스트 결과 채움 → **운영↔개발 AI 비교 리포트 화면 출력**까지 한 번의 시연으로 완주.

- **1차 대상은 실제 운영/개발 소개 사이트**(공개·민감정보 없음 → 외부 AI 전송 무방).
- **mock 사이트(`:10002`/`:10003`)는 데모 안전망** — 실제 사이트가 네트워크·응답 문제로 막혀도 동일 골든 패스를 mock으로 시연 가능.
- AI 키가 없거나 실패해도 건강성 점검·결과 테이블까지는 **항상** 보여준다(graceful degradation).
- 이 골든 패스 위의 기능이 **최우선**이며, 그 밖은 "있으면 좋은 것"으로 둔다.

---

## 3. 사용자 시나리오

| 단계 | 사용자 행동 | 시스템 반응 | 관련 탭/화면 |

|---|---|---|---|

| 1 | 시스템 등록(이름·설명·환경 URL들) | 환경별 크롤을 **동시** 스케줄 | `#/systems/new` → SystemForm |

| 2 | 진행 확인 | 크롤 status가 pending→running→done | overview / sitemap |

| 3 | 사이트 구조 확인 | 경로 트리, 노드 클릭 시 상세 | sitemap (NodeDetailPanel) |

| 4 | 테스트케이스 생성/확인 | 크롤 페이지에서 TC 자동 분류·생성 | test-cases |

| 5 | 자동/AI 테스트 실행 | 건강성 점검(+키 있으면 AI 평가), 진행률 폴링 | runs / results |

| 6 | 결과·리포트 확인 | URL별 status·상세, 집계 | results / reports |

| 7 | 운영↔개발 비교 | 경로 집합 AI 비교, 증적 리포트 | compare |

| 8 | 수동 보정 | 특정 URL status/메모 덮어쓰기 | sitemap 상세 패널 |

---

## 4. 데이터 모델

**SQLite 단일 파일** `backend/test_support.db`, SQLAlchemy 2.0(`models.py`).

```

System 1—N Environment

Environment 1—1 CrawlResult

Environment 1—1 TestRun

Environment 1—N TestResult        ← (environment_id, url) 키, 재크롤/재실행에도 보존

System 1—1 CompareRun
```

### 테이블 / 컬럼

**System** — `id, name, description, created_at`

**Environment** — `id, system_id(FK), name, url, render_js(bool, JS 렌더 크롤 여부), crawl_depth(int|null), max_pages(int|null)`

> `crawl_depth/max_pages`가 null이면 크롤러 기본값 사용. 환경별 크롤 한도 오버라이드.

**CrawlResult** — `id, environment_id(FK), status, tree_json, page_count, error, crawled_at`

> `status ∈ {pending, running, done, error}`. `tree_json`이 사이트 구조 트리.

**TestRun** (1회 실행/집계) — `id, environment_id(FK), status, error, ai_enabled,`

`pages_total, done_count, pass_count, fail_count, blocked_count, ai_summary_json, started_at, finished_at`

> 진행률 폴링용 집계. `ai_enabled`는 이번 실행에서 Claude 평가가 돌았는지.

**TestResult** (URL별 결과 저장소) — `id, environment_id(FK), url, status, note, tester, detail_json, updated_at`

> `status ∈ {pass, fail, blocked, untested}`. `tester`는 `"AI"` 또는 사람 이름.

> `detail_json`에 자동 점검 상세(http_status, load_ms, issues[], ai{}). **이 저장소가 사이트맵/결과/리포트의 단일 소스.**

**CompareRun** (운영↔개발 증적) — `id, system_id(FK), status, error, prod_env, dev_env, report_json, started_at, finished_at`

> `report_json`에 comparator 리포트 전체(썸네일 포함).

### status enum 요약

| 영역 | 값 |

|---|---|

| 크롤/테스트/비교 진행 | `pending` · `running` · `done` · `error` (API 상태에선 `idle`/`scheduled` 추가) |

| URL별 테스트 결과 | `pass` · `fail` · `blocked` · `untested` |

### 마이그레이션

정식 도구 없음. `main.py::_migrate()`가 기존 DB에 누락 컬럼(예: `environments.render_js`,

`crawl_depth`, `max_pages`, TestRun 집계 컬럼)을 가볍게 `ALTER` 한다.

---

## 5. API 명세

모든 경로는 `/api` 프리픽스. 백엔드 FastAPI `:10001`, 문서 `/docs`. 비동기 작업은 `BackgroundTasks`.

스키마는 `schemas.py`의 Pydantic 모델 기준.

### 시스템 CRUD

| 메서드 | 경로 | 상태 | 요청 | 응답 |

|---|---|---|---|---|

| GET | `/api/systems` | 200 | — | `list[SystemOut]` |

| POST | `/api/systems` | 201 | `SystemCreate` | `SystemOut` (+ 환경별 크롤 동시 스케줄) |

| GET | `/api/systems/{id}` | 200 | — | `SystemOut` |

| PUT | `/api/systems/{id}` | 200 | `SystemUpdate` | `SystemOut` |

| DELETE | `/api/systems/{id}` | 204 | — | — |

- **POST**: 등록 직후 `run_crawls_concurrently`로 환경별 크롤을 동시 실행.
- **PUT**: URL/render 옵션이 **바뀐 환경**과 **신규 환경**만 재크롤. `id` 빠진 환경은 cascade 삭제.
- `SystemCreate { name, description="", environments: [EnvironmentCreate] }`
- `EnvironmentCreate/Update { (id?), name, url, render_js=false, crawl_depth?, max_pages? }`

### 크롤

| 메서드 | 경로 | 상태 | 응답 |

|---|---|---|---|

| POST | `/api/environments/{id}/crawl` | 202 | `{status:"scheduled"}` |

| GET | `/api/environments/{id}/crawl` | 200 | `CrawlDetail { status, page_count, error, crawled_at?, tree? }` |

> `render_js=true`면 `crawl_site_rendered`(Playwright), 아니면 `crawl_site`(정적). 결과 → `build_tree` → `tree_json`.

### 테스트 (URL별 결과 저장소)

| 메서드 | 경로 | 상태 | 요청 | 응답 |

|---|---|---|---|---|

| POST | `/api/environments/{id}/test` | 202 | — | `{status:"scheduled", ai_available}` (크롤 미완료 시 **409**) |

| GET | `/api/environments/{id}/test` | 200 | — | `TestRunStatus`(진행률 폴링) |

| GET | `/api/environments/{id}/tests` | 200 | — | `TestResultsResponse { results:[TestResultOut] }` |

| PUT | `/api/environments/{id}/tests` | 200 | `TestResultUpsert` | `TestResultOut` (수동 덮어쓰기) |

| POST | `/api/environments/{id}/tests/run-one` | 200 | `RunOneRequest { url }` | `TestResultOut` (단건 즉시 실행) |

- `run_test`: 크롤 트리에서 페이지 수집 → `health_check`(키 불필요) → 키 있으면 상위 `AI_PAGE_LIMIT`(기본 15)

  페이지를 `evaluate_page`로 AI 평가(동시성 Semaphore 4) → `TestResult` upsert. 건강성 status(ok/warn/fail)를

  프론트 status(pass/blocked/fail)로 환산.
- `TestRunStatus { status, error, ai_enabled, ai_available, pages_total, done_count, pass_count, fail_count, blocked_count, started_at?, finished_at?, ai_summary? }`
- `TestResultUpsert { url, status="untested"∈{pass,fail,blocked,untested}, note="", tester="" }` — status 검증 validator 존재.
- `TestResultOut { url, status, note, tester, updated_at?, detail? }`

### 비교 (운영↔개발)

| 메서드 | 경로 | 상태 | 응답 |

|---|---|---|---|

| POST | `/api/systems/{id}/compare` | 200 | 동기 AI 비교 리포트(키 필수) |

| POST | `/api/systems/{id}/compare-run` | 202 | `{status:"scheduled"}` (비동기 증적) |

| GET | `/api/systems/{id}/compare-run` | 200 | `CompareRunStatus { status(idle/pending/running/done/error), error, prod_env, dev_env, started_at?, finished_at?, report? }` |

> 분석 완료된 환경 2개의 경로 집합을 `compare_environments`로 AI 비교.
>
> **표준 경로 = `compare-run`(비동기).** 발표 골든 패스는 `POST /compare-run` → `GET /compare-run` 폴링으로
> `CompareRun.report_json`(썸네일 포함) 증적을 화면에 띄운다.
> 동기 `compare`는 단건 즉시 확인용 **보조**이며 **발표 동선엔 미사용**.

### 헬스

| 메서드 | 경로 | 응답 |

|---|---|---|

| GET | `/api/health` | `{status:"ok"}` |

---

## 6. AI 동작 명세

모듈 `ai_tester.py` / `comparator.py`. 키는 `backend/.env`의 `ANTHROPIC_API_KEY`, 모델 기본값 `claude-opus-4-8`(`ANTHROPIC_MODEL`로 변경).

| 기능 | 키 필요 | 설명 |

|---|---|---|

| `health_check` | ❌ | HTTP 상태·깨진 링크/이미지·alt 누락, `render_js`면 콘솔 에러 수집 |

| `evaluate_page` | ✅ | 상위 `AI_PAGE_LIMIT`(기본 15) 페이지를 Claude 평가. 동시성 Semaphore 4 |

| `compare_environments` | ✅ | 운영↔개발 경로 집합 비교 |

- **Graceful degradation**: 키가 없어도 건강성 점검은 항상 동작한다. AI 평가/비교만 비활성.

  `ai_available()`가 키 보유 여부를 알려주고 테스트 트리거 응답에 포함된다.
- Claude 응답은 `_parse_json`으로 코드펜스/잡텍스트를 벗겨 파싱.

---

## 7. 프론트엔드 / UI 명세

**React + Vite, 라우터 라이브러리 없음.** `App.jsx::parseRoute`가 `window.location.hash`를 직접 파싱하는 해시 라우팅.

모든 백엔드 호출은 `src/api.js`의 `api` 객체에 집중(`/api` 상대경로 → Vite 프록시).

### 해시 라우팅 맵

| 해시 | 화면 |

|---|---|

| `#/` | 랜딩 |

| `#/dashboard` | 포털 대시보드(전체 시스템) |

| `#/systems` · `#/systems/new` | 시스템 목록 · 신규 등록 |

| `#/systems/{id}` · `#/systems/{id}/{tab}` | 시스템 상세(기본 탭 overview) |

| `#/history` · `#/reports` · `#/settings` · `#/architecture` | 실행 이력 · 리포트 · 전역 설정 · 아키텍처 |

### 시스템 상세 탭 (8종)

| 탭 | 그룹 | 역할 |

|---|---|---|

| overview | work | 시스템 요약·상태 |

| sitemap | work | 크롤 트리, 노드 상세(NodeDetailPanel), 수동 결과 보정 |

| test-cases | work | 크롤 페이지 → 테스트케이스 자동 생성/분류·필터 |

| runs | work | 테스트 실행 이력 |

| results | analysis | URL별 결과 테이블 |

| compare | analysis | 운영↔개발 비교 증적 |

| reports | analysis | 리포트 집계 |

| settings | config | 시스템 설정 |

### 디자인 시스템 (NHQA 브랜드)

`src/styles.css`의 CSS 커스텀 프로퍼티 토큰 + `.design-sync`로 큐레이션된 7개 프리미티브.

**핵심 토큰**

| 용도 | 변수 | 값 |

|---|---|---|

| 브랜드 그린(농협) | `--color-primary` / `--color-brand` | `#00A651` |

| QA/IT 블루 | `--color-accent-blue` | `#2563EB` |

| 성공/경고/위험 | `--color-success` / `--color-warning` / `--color-danger` | `#16B364` / `#D97706` / `#DC2626` |

| 환경색 PROD | `--color-env-prod` | `#6D28D9` (보라) |

| 환경색 DEV | `--color-env-dev` | `#2563EB` (블루) |

| 라운드/그림자 | `--radius` / `--shadow-md` | `14px` / `0 10px 24px rgba(15,23,42,.06)` |

폰트: `"Inter", -apple-system, …`. 모노: `"SF Mono", ui-monospace, Menlo`.

**재사용 프리미티브** (`.design-sync/config.json`에 등록, `window.NHDS`로 노출)

| 컴포넌트 | 주요 props |

|---|---|

| `Badge` | `status?`(pass/fail/warning/blocked/pending/untested/running/done/error), `env?`(PROD/DEV/ENV), `dot?` |

| `EnvironmentBadge` | `env?{name,url}`, `kind?`(자동판별) |

| `StatCard` | `label, value, icon?, tone?(muted/success/warning/danger), progress?` |

| `StatusSummaryBanner` | `failCount?, rate?` |

| `SummaryCards` | `items:[{label,value,tone?,small?}]` |

| `EmptyState` | `icon?, title?, description?, actionLabel?, onAction?, secondary?, compact?` |

| `NHLogo` | `size?`(기본 30), `variant?:'lockup'` — 에셋 `/public/nhqa-logo.png` |

**필터/컨트롤**: `filters/SegmentedControl`(←/→·Enter), `filters/FilterDropdown`(단일/다중), `SortHeader`(asc/desc 토글).

### 상태 매핑

| 결과 status | 배지 클래스 | 색 |

|---|---|---|

| pass / done | `.badge-pass` | 초록 |

| fail / error | `.badge-fail` | 빨강 |

| blocked / warning | `.badge-warning` | 앰버 |

| untested / pending | `.badge-pending` | 회색(pending은 펄스) |

| running | `.badge-running` | 초록(펄스) |

환경 자동판별(`lib/qa.js::envKind`): name/url에 `운영·prod·live·상용` → **PROD**, `개발·dev·stag·qa·test·local` → **DEV**, 그 외 **ENV**.

### lib 유틸 (재사용 — React 비의존 순수 함수)

| 파일 | 주요 export |

|---|---|

| `lib/qa.js` | `envKind`, `rateTone`, `aggregateTests`, `sumAggregates`, `flattenPages` |

| `lib/result.js` | `httpClass`, `msInfo`, `severityClass`, `urlPath`, `urlHost`, `compareValues` |

| `lib/datetime.js` | `parseServerDate`, `fmtDateTime`, `fmtDate`, `relativeTime` (KST 기준) |

| `lib/url.js` | `withScheme`("naver.com"→"https://naver.com") |

| `lib/settings.js` | `crawlDefaults`, localStorage 키 `qa-portal-settings` |

| `lib/testcases.js` | `classifyTestCaseType`, `getTestCasePriority`, `generateTestCasesFromCrawledPages`, `filterTestCases` (TC 타입 FUNCTION/UI/INPUT/EXCEPTION/AUTH/API, 우선순위 P0~P3) |

---

## 8. 기술적 접근 / 모듈 역할

### 백엔드 모듈

- `crawler.py` — same-site 판정(`tldextract` 등록 도메인, 서브도메인 허용 / localhost·IP는 netloc 완전 일치),

  `normalize_url`(쿼리·프래그먼트 제거), robots.txt+sitemap.xml 시드, BFS(`max_pages=80, max_depth=3`),

  `build_tree`(경로 세그먼트 트리, 멀티 호스트면 호스트별 묶음). `tldextract`는 번들 스냅샷만 써 오프라인 동작.
- `crawler_render.py` — Playwright(Chromium) 렌더 후 링크 수집(느림, 기본 max 40·depth 2).
- `ai_tester.py` — `collect_pages`, `health_check`, `evaluate_page`, `compare_environments`, `_parse_json`.
- `comparator.py` — 운영↔개발 증적 리포트 생성(썸네일 포함).
- `database.py` — engine/SessionLocal/Base/get_db.
- `main.py` — 라우트 + `_migrate()` + 백그라운드 작업(`run_crawl`, `run_crawls_concurrently`, `run_test`, compare).

### 백그라운드 작업 관례

- 새 `SessionLocal()`을 열고 `finally`에서 닫는다(요청 세션과 분리).
- 실패는 예외를 던지지 않고 해당 레코드의 `status="error"` + `error`에 기록해 노출한다.

### 포트 규약 (반드시 준수)

이 앱은 **10000~10010만** 사용하고 `0.0.0.0`에 바인딩한다(외부 오픈).

| 서비스 | 포트 |

|---|---|

| Frontend (Vite) | `:10000` (`/api` → `:10001` 프록시) |

| Backend (FastAPI) | `:10001` (`/docs`) |

| Mock 운영 / 개발 | `:10002` / `:10003` (`/nh` = 농협정보시스템 데모) |

`mock-sites/serve.py`는 stdlib `ThreadingHTTPServer`로 데모 사이트를 띄운다(백엔드 venv 파이썬, 3.7+).

`./run.sh`가 셋을 한 번에 띄운다.

---

## 9. 영향 / 리스크

- **외부 오픈 바인딩**: `0.0.0.0` + 인증 없음 → 신뢰된 네트워크에서만 운영할 것.
- **SQLite 단일 파일**: 동시 쓰기/규모 한계. 백업은 파일 복사.
- **CORS**: 화이트리스트가 `5173`만 열려 있으나 브라우저는 Vite 프록시로 동일 출처가 되어 무방. 직접 호출 코드 추가 시 CORS 손봐야 함.
- **Playwright 크롤 비용**: `render_js=true`는 느리고 무겁다. 기본은 정적 크롤.
- **AI 비용/속도**: `evaluate_page`는 상위 N페이지로 제한(Semaphore 4)하지만 키·토큰 비용 발생.
- **크롤 매너**: robots.txt/sitemap.xml 시드를 따른다.

---

## 10. 검증 방법

```bash

# 백엔드 테스트

cd backend && source .venv/bin/activate && python -m pytest -q

python -m pytest test_crawler.py::<test_name>   # 단일


# 키 없이 전체 흐름 시연 (mock 사이트 활용)

./run.sh                       # 10000/10001/10002/10003 기동

#  → 시스템 등록(운영 :10002, 개발 :10003) → 크롤 done → 테스트(건강성만) → 결과 확인

#  → GET /api/health 로 백엔드 확인, /docs 로 API 확인
```

- **API**: `/docs`(Swagger)에서 엔드포인트·스키마 일치 확인.
- **AI 경로**: `backend/.env`에 `ANTHROPIC_API_KEY` 설정 시 AI 평가·비교 동작 확인.
- **프론트 수동 시나리오**: §3 표의 8단계를 탭 따라 진행.

---

## 11. 미해결 질문

- **인증 부재**: 현재 사용자/권한 모델이 없다. 사내 배포 시 접근 제어가 필요한지.
- **테스트 커버리지**: `test_crawler.py` 외 백엔드/프론트 테스트 범위가 어디까지인지(크롤 위주로 보임).
- `detail_json` / `ai_summary_json` / `report_json`의 정확한 내부 스키마는 코드에서 동적으로 구성되어 본 문서엔 키 수준만 기재.

> **[결정됨] 비교 경로**: 표준은 비동기 `/compare-run`. 동기 `/compare`는 보조(발표 미사용). → §5 반영.
>
> **[결정됨] 포트**: 정본은 `run.sh`/`vite.config.js` 기준 **10000~10003**(§8 포트 규약). README의 8000/5173은 구버전 오기 → README를 10000/10001로 수정할 것.

```
```
