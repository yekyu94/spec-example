# PRD: AI 평가·비교 레이어 활성화 + 골든 패스 안정화

Status: ready-for-agent

> 대상 시스템: 테스트 지원 포털(Test Support Portal). 본 PRD는 `spec.md`(as-built 명세), `feedback-synthesis.md`(완주 가능성 종합), `tech-verification.md`(기술 검증)를 종합해 작성됨. 7/15 해커톤 Definition of Done 완주를 목표로 한다.

## Problem Statement

QA 담당자(사내 다수 시스템을 시스템별로 나눠 맡은 여러 명)는 운영(PROD)·개발(DEV) 환경에 흩어진 웹 시스템을 수동 체크리스트로 점검하고, 운영과 개발이 동일하게 배포됐는지 눈으로 비교한다. 느리고 누락이 많다.

테스트 지원 포털은 이 흐름(등록 → 크롤 → URL별 자동/AI 테스트 → 운영↔개발 비교)을 이미 자동화했고, **LLM 연동을 제외한 전 경로(크롤·건강성 점검·사이트맵·수동 보정·결과 저장소)가 동작한다.** 그러나 발표 골든 패스의 핵심 가치인 **"AI가 페이지를 평가하고 운영↔개발을 비교해 증적 리포트를 만든다"** 부분이 아직 살아 있지 않다. API 키가 곧 발급되지만, 키를 꽂는 것만으로는 끝이 아니다 — opus-4-8의 API 제약(금지 파라미터, JSON 파싱 안정성, 대용량 출력)과 닫히지 않은 데모 경로 결정(`compare` vs `compare-run`, 포트/README 불일치)이 시연 당일 발목을 잡을 수 있다.

요컨대: **AI 평가·비교 레이어를 키 발급 후 실제로 "쓸 만하게" 동작시키고, 골든 패스 한 줄기를 끊김 없이 돌게 안정화해야 한다.**

## Solution

키 발급을 전제로 AI 평가(`evaluate_page`)·비교(`compare_environments` via `compare-run`) 레이어를 opus-4-8에서 안정적으로 동작시키고, 골든 패스(운영·개발 둘 다 등록 → 크롤 done → URL별 자동/AI 테스트 → 운영↔개발 AI 비교 리포트 화면 출력)를 한 번의 시연으로 완주 가능하게 만든다.

사용자 관점에서:

- QA 담당자는 시스템과 운영·개발 URL을 등록하면, 키가 있을 때 상위 페이지가 자동으로 AI 평가되고 운영↔개발 비교 증적 리포트가 화면에 뜬다.
- 키가 없거나 AI 호출이 실패해도 건강성 점검·결과 테이블까지는 **항상** 보이며(graceful degradation), 화면이 키 보유 여부(`ai_available`)를 명확히 알려준다.
- 발표자는 실제 운영/개발 소개 사이트로 시연하되, 네트워크·응답 문제 시 동일 골든 패스를 mock 사이트(`:10002`/`:10003`)로 그대로 재현할 수 있다.

## User Stories

1. As a QA 담당자, I want 시스템 등록 시 운영·개발 URL을 함께 입력하면 환경별 크롤이 동시에 시작되기를, so that 두 환경 구조를 한 번에 빠르게 확보한다.
2. As a QA 담당자, I want 크롤 진행 상태(pending→running→done→error)를 폴링으로 보기를, so that 언제 테스트를 돌릴 수 있는지 안다.
3. As a QA 담당자, I want 크롤이 완료된 환경에서만 테스트가 시작되고 미완료 시 명확히 거부(409)되기를, so that 빈 트리에 헛돈을 들이지 않는다.
4. As a QA 담당자, I want API 키가 없어도 건강성 점검(HTTP 상태·깨진 링크/이미지·alt 누락)이 항상 동작하기를, so that 키 발급 전·실패 시에도 결과를 얻는다.
5. As a QA 담당자, I want 화면이 키 보유 여부(`ai_available`)를 명확히 표시하기를, so that AI 결과를 기대해도 되는지 사전에 안다.
6. As a QA 담당자, I want 키가 있으면 상위 N개(기본 15) 페이지가 자동으로 Claude 평가되기를, so that 사람이 페이지마다 수동 점검하지 않아도 된다.
7. As a QA 담당자, I want AI 평가가 동시성 제한(Semaphore 4) 아래 돌기를, so that 비용·속도가 통제되고 레이트리밋에 걸리지 않는다.
8. As a QA 담당자, I want AI 평가 결과가 URL별 결과 저장소(`TestResult`)에 `tester="AI"`로 저장되기를, so that 자동·수동 결과가 한 저장소에서 공존한다.
9. As a QA 담당자, I want 건강성 status(ok/warn/fail)가 프론트 status(pass/blocked/fail)로 일관 환산되기를, so that 결과 테이블·배지가 한 색 체계로 읽힌다.
10. As a QA 담당자, I want 특정 URL의 status/메모를 수동으로 덮어쓰기를, so that AI·자동 판정이 틀린 경우 보정한다.
11. As a QA 담당자, I want 단건 URL을 즉시 재실행(run-one)하기를, so that 전체 재실행 없이 한 페이지만 다시 확인한다.
12. As a QA 담당자, I want 재크롤/재실행해도 URL별 결과가 보존되기를, so that 누적된 점검 이력이 사라지지 않는다.
13. As a QA 담당자, I want 운영↔개발 비교를 비동기(`compare-run`)로 돌리고 진행 상태를 폴링하기를, so that 긴 AI 비교 동안 화면이 멈추지 않는다.
14. As a QA 담당자, I want 비교가 완료되면 증적 리포트(경로 집합 차이, 썸네일 포함)를 화면에서 보기를, so that 운영과 개발이 동일 배포인지 한눈에 판단한다.
15. As a QA 담당자, I want AI 비교가 키 없이 호출되면 명확한 안내를 받기를, so that 왜 비교가 안 되는지 혼란이 없다.
16. As a QA 담당자, I want AI 호출이 실패해도 해당 레코드가 `error` status와 사유로 노출되기를(예외로 전체가 죽지 않고), so that 어디서 막혔는지 알고 나머지 결과는 그대로 본다.
17. As a QA 담당자, I want AI가 반환한 JSON이 코드펜스·잡텍스트에 섞여 와도 안정적으로 파싱되기를, so that 모델 출력 형식 변동으로 결과가 깨지지 않는다.
18. As a 발표자, I want 표준 데모 경로가 `compare-run`(비동기) 하나로 고정되기를, so that 시연 중 어느 버튼을 누를지 헷갈리지 않는다.
19. As a 발표자, I want 포트(10000~10003)와 README가 일치하기를, so that `./run.sh` 한 번으로 전 스택이 예측대로 뜬다.
20. As a 발표자, I want 실제 사이트가 막히면 mock 사이트로 동일 골든 패스를 재현하기를, so that 네트워크 사고가 시연을 멈추지 않는다.
21. As a 개발자, I want AI 평가/비교 코드가 opus-4-8에서 금지된 파라미터(`temperature`/`top_p`/`top_k`/`budget_tokens`)를 보내지 않기를, so that 400 에러로 AI 경로가 통째로 죽지 않는다.
22. As a 개발자, I want 큰 AI 출력(>~16K 토큰)이 스트리밍으로 처리되기를, so that 비교 리포트가 길어져도 잘리거나 타임아웃되지 않는다.
23. As a 개발자, I want 실제 키 없이도 AI 경로의 두 분기(키 있음/없음)를 테스트할 수 있기를, so that CI·로컬에서 비용 없이 동작을 검증한다.
24. As a QA 담당자, I want AI 평가 결과의 품질(프롬프트가 의미 있는 평가를 내는지)을 키 발급 후 별도로 확인하기를, so that "키만 꽂으면 끝"이라는 낙관에 빠지지 않는다.

## Implementation Decisions

**대상 모듈 (수정/활성화)**

- `ai_tester.py` — `collect_pages`, `health_check`(키 불필요), `evaluate_page`(키 필요), `compare_environments`(키 필요), `_parse_json`. AI 호출 파라미터를 opus-4-8 제약에 맞게 정리한다.
- `comparator.py` — 운영↔개발 증적 리포트 생성(썸네일 포함). `compare-run` 비동기 경로의 백엔드 산출물.
- `main.py` — 백그라운드 작업(`run_test`, compare-run), `ai_available()` 노출, 라우트 동작은 유지. 데모 경로를 `compare-run`으로 고정.

**AI 호출 규약 (opus-4-8 제약 반영 — tech-verification 기준)**

- `temperature` / `top_p` / `top_k` / `budget_tokens` 파라미터를 **전송하지 않는다.** opus-4-8는 이들 전달 시 400을 반환한다. 사고 제어가 필요하면 `thinking={"type":"adaptive"}`(adaptive thinking)로 전환한다.
- 모델 ID는 `claude-opus-4-8`(날짜 접미사 없음). 키는 `backend/.env`의 `ANTHROPIC_API_KEY`, 모델 오버라이드는 `ANTHROPIC_MODEL`. `anthropic.Anthropic()`이 키를 자동으로 읽는 SDK 관례를 따른다.
- 큰 출력(>~16K 토큰, 특히 비교 리포트)은 **스트리밍**으로 받는다. opus-4-8 최대 출력 128K는 스트리밍 전제.
- JSON 파싱 안정화: 현행 `_parse_json`(코드펜스/잡텍스트 제거)을 유지하되, 안정성 강화를 위해 Structured Outputs(`output_config.format` 또는 `client.messages.parse()` + Pydantic) 채택을 우선 검토한다. 단, 본 PRD의 합격선은 "키 발급 후 AI 경로가 끊김 없이 동작"이며, Structured Outputs 전환은 그 안에서의 신뢰성 개선 수단이다.

**Graceful degradation 계약 (불변)**

- 키 부재 시: `health_check`는 항상 동작하고, `evaluate_page`/`compare_environments`만 비활성. `ai_available()`가 키 보유 여부를 반환하고 테스트 트리거 응답(`{status:"scheduled", ai_available}`)에 포함된다.
- AI 호출 실패는 예외를 전파하지 않고 해당 레코드의 `status="error"` + `error` 필드에 기록해 노출한다(백그라운드 작업 관례).

**데이터/저장소 계약 (유지)**

- 단일 소스는 `(environment_id, url)` 키의 `TestResult`. 자동(`tester="AI"`)·수동 결과가 공존하며 재크롤/재실행에도 보존된다.
- 건강성 status(ok/warn/fail) → 프론트 status(pass/blocked/fail) 환산 규칙 유지. URL별 status enum: `pass`·`fail`·`blocked`·`untested`.
- 진행률 폴링은 `TestRunStatus`(pages_total/done/pass/fail/blocked + `ai_enabled`/`ai_available`)로 노출.

**비교 경로 단일화 (열린 결정 닫기)**

- 표준 = 비동기 `POST /api/systems/{id}/compare-run` → `GET .../compare-run` 폴링 → `CompareRun.report_json`(썸네일 포함)을 화면에 출력. 동기 `POST /compare`는 단건 보조로만 남기고 **발표 동선에서 제외**한다.

**환경/실행 규약 (열린 결정 닫기)**

- 포트는 10000~10003 고정(Frontend `:10000` → `/api` `:10001` 프록시, Backend `:10001`, Mock 운영/개발 `:10002`/`:10003`). README의 8000/5173 구버전 오기를 10000/10001로 정정한다.
- `./run.sh`가 4개 서비스를 한 번에 기동하는 동선을 표준 시연 경로로 삼는다.

## Testing Decisions

**좋은 테스트의 기준.** 외부에서 관찰 가능한 동작만 검증한다 — 내부 함수 시그니처·구현 디테일이 아니라, HTTP 응답의 상태코드·스키마·status enum 전이를 본다. AI 모델의 실제 출력 내용(품질)은 자동 테스트의 대상이 아니라 키 발급 후 사람이 확인하는 별도 작업이다(스토리 24).

**심(seam) — 합의됨.**

- **주 심: 백엔드 HTTP API 경계** (`/api/...`, FastAPI TestClient). 골든 패스 전 구간을 엔드포인트 레벨에서 검증한다: 시스템 등록 → 크롤 status 전이 → 크롤 미완료 시 테스트 409 → 테스트 실행 → `TestResult` 저장/조회/수동 덮어쓰기/run-one → `compare-run` 스케줄·폴링·리포트.
- **외부 의존성 심: Anthropic 클라이언트 단일 주입 지점 모킹.** 실제 키 없이 가짜 Claude 응답을 주입해, AI 경로의 두 분기(`ai_available`=true/false)와 실패 시 `error` 기록·graceful degradation을 모두 검증한다. 심을 코드베이스 전반에 흩지 않고 API 경계 1 + 외부 모킹 경계 1로 수렴한다.

**테스트 대상 모듈.**

- API 경계 전반(systems CRUD, crawl, test, tests, run-one, compare-run, health).
- `ai_tester.py`/`comparator.py`는 직접 단위 테스트하지 않고, 모킹된 Anthropic 클라이언트를 통해 API 경계에서 간접 검증한다(심 최소화 원칙).

**선례(prior art).** `backend/test_crawler.py`(pytest). 신규 API 테스트는 동일 pytest 관례·픽스처 스타일을 따른다. 검증 명령: `cd backend && source .venv/bin/activate && python -m pytest -q`.

**핵심 검증 시나리오.**

- 키 없음: 테스트 트리거 응답 `ai_available=false`, 건강성 결과만 채워지고 AI 필드는 비어도 200/정상 status.
- 키 있음(모킹): 상위 N 페이지에 AI 평가가 `tester="AI"`로 저장, 진행률 집계 정확.
- AI 호출 실패(모킹 예외): 레코드 `status="error"` + 사유, 나머지 결과는 보존.
- compare-run: 분석 완료 환경 2개에서 스케줄(202) → 폴링 status 전이 → `report` 반환.
- 금지 파라미터 회귀 방지: AI 호출 시 `temperature`/`top_p`/`top_k`/`budget_tokens`가 전달되지 않음을 모킹 클라이언트 호출 인자로 검증.

## Out of Scope

- **새 기능 추가.** 이 시스템은 이미 동작하는 작품이며, 본 PRD는 "더 만들기"가 아니라 "완주·전달"이 목표다. 골든 패스 밖 기능은 "있으면 좋은 것"으로 둔다.
- **인증/권한/멀티테넌시.** 로그인·사용자 모델·접근 제어는 도입하지 않는다(`0.0.0.0`+무인증은 신뢰된 네트워크 전제로 유지, §9 리스크 그대로).
- **외부 CI/이슈 트래커 연동.** 결과는 SQLite 단일 파일에만 저장.
- **정식 마이그레이션 도구(Alembic 등).** `main.py::_migrate()`의 수작업 ALTER 유지.
- **AI 출력 품질의 자동 검증.** 프롬프트가 의미 있는 평가를 내는지는 키 발급 후 사람이 확인하는 작업이며, 자동 테스트 합격선이 아니다.
- **`render_js=true` 렌더 크롤의 성능 최적화.** 기본은 정적 크롤. 렌더 크롤은 비용이 크므로 시연 표준 경로에서 제외.

## Further Notes

- **이 단계 최대 함정:** "시간 남으니 기능 더 넣자" → 데모가 깨진다. 평가 비중이 큰 완성도(끝까지 도는가)·발표(문제→해결 전달)에 남은 시간을 쓴다. (피드백 양쪽 공통)
- **"키만 꽂으면 끝"은 낙관.** 키 연동과 AI 출력 품질은 별개 작업. 키 발급(오늘 예정) 후 *실제 평가 결과가 쓸 만한지* 검증할 시간을 따로 잡는다. (스토리 24)
- **사용자(Who) 서술 보강 여지:** "어떤 QA가 지금 어떤 환경에서 어떻게 일하는지" 한두 줄을 발표 자료에 더하면 전달력이 오른다. (PRD 범위 밖이나 발표 준비 메모로 남김)
- **팀 맥락:** 3명(LLM 연동 경험자 보유), 하루 2~3시간(3주 누적 ≈ 100시간), 외부 API 호출 가능(`api.anthropic.com` 사내망 미차단). 범위가 안정화로 좁아 완주 여력은 충분하다는 것이 종합 피드백의 결론.
- **출처:** `spec.md` §1~§11, `feedback-synthesis.md`, `tech-verification.md`. opus-4-8 제약은 `claude-api` 공식 레퍼런스 검증 기준.
