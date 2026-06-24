# 테스트 지원 포털 — 기술 스택 검증 보고서

> 작성일: 2026-06-24
> 대상: `feedback-synthesis.md` / `spec.md` 기준 기술 스택
> 검증 도구: context7(오픈소스 문서), `claude-api` 공식 레퍼런스(Anthropic)

---

## 1. 식별된 기술 스택

`spec.md` 기준으로 시스템이 사용하는 기술을 분류했다.

### 백엔드
| 기술 | 용도 |
|---|---|
| FastAPI | API 서버(`:10001`), `BackgroundTasks`로 비동기 크롤/테스트 |
| SQLAlchemy 2.0 | ORM (`models.py`), SQLite 단일 파일 |
| Pydantic | 요청/응답 스키마 (`schemas.py`) |
| SQLite | 단일 파일 저장소 `test_support.db` |
| Playwright (Python) | JS 렌더 크롤 (`crawler_render.py`, Chromium) |
| tldextract | same-site 판정 (등록 도메인) |
| pytest | 백엔드 테스트 |
| Anthropic Claude (`claude-opus-4-8`) | AI 평가/비교 (`ai_tester.py`, `comparator.py`) |
| stdlib ThreadingHTTPServer | mock 사이트 서버 (`mock-sites/serve.py`) |

### 프론트엔드
| 기술 | 용도 |
|---|---|
| React + Vite | SPA(`:10000`), 해시 라우팅(라우터 라이브러리 없음) |
| CSS 커스텀 프로퍼티 | NHQA 디자인 토큰 (`styles.css`) |

---

## 2. 검증 결과

### ✅ Claude / Anthropic (`claude-opus-4-8`) — 검증 완료

- **`claude-opus-4-8`은 현존하는 정확한 모델 ID.** Anthropic의 최상위 Opus급 모델(컨텍스트 1M, 출력 최대 128K). 날짜 접미사를 붙이면 안 되며 spec 표기가 정확하다.
- spec의 환경변수 패턴(`ANTHROPIC_API_KEY` + `ANTHROPIC_MODEL` 오버라이드)은 Python SDK 관례와 일치 — `anthropic.Anthropic()`이 `ANTHROPIC_API_KEY`를 자동으로 읽는다.

**개선 포인트**
- `_parse_json`(코드펜스/잡텍스트 제거 후 파싱)은 동작하지만, opus-4-8에서는 **Structured Outputs**(`output_config.format` 또는 `client.messages.parse()` + Pydantic)를 쓰면 JSON 파싱 안정성이 보장된다. → feedback-synthesis의 "`_parse_json` 안정성" 우려를 구조적으로 해소.
- opus-4-8는 `budget_tokens` / `temperature` / `top_p` / `top_k` 전달 시 **400 에러**. AI 평가 코드가 이 파라미터를 쓰고 있다면 제거하고 **adaptive thinking**(`thinking={"type":"adaptive"}`)으로 전환해야 한다.
- 큰 출력(>~16K 토큰)은 스트리밍 필요. opus-4-8는 최대 128K 출력 가능하나 스트리밍 전제.

### ✅ FastAPI — 라이브러리 확인

- context7에서 라이브러리 확인됨 (`/websites/fastapi_tiangolo`, 신뢰도 High, 코드 스니펫 5270개).
- `BackgroundTasks` 기반 비동기 작업, `/api` 프리픽스 라우팅, 201/202/204 상태코드, Pydantic 응답 모델은 모두 FastAPI 표준 기능으로 spec 기술과 부합.
- (상세 문서 페이지는 context7 분당 호출 한도로 미열람 — 한도 해제 후 확인 가능)

### ⏳ 미확인 (context7 분당 한도 초과)

| 기술 | 상태 |
|---|---|
| SQLAlchemy 2.0 | resolve 실패 (rate limit) |
| Playwright (Python) | resolve 실패 (rate limit) |
| Pydantic / tldextract / pytest | 식별만 완료, 문서 미조회 |

> context7 무료 한도(분당 호출 제한)에 걸려 약 46분간 추가 조회가 차단됨.

---

## 3. 결론 및 권고

1. **핵심 위험 요소(Claude 연동)는 검증됨** — 모델 ID·환경변수·키 없는 graceful degradation 설계가 모두 타당.
2. **즉시 점검 권장:** `ai_tester.py` / `comparator.py`가 opus-4-8에서 금지된 파라미터(`temperature`, `budget_tokens` 등)를 쓰는지 확인. 쓰고 있다면 제거.
3. **선택적 개선:** AI 출력 파싱을 Structured Outputs로 전환해 `_parse_json` 의존도 축소.
4. **후속:** context7 한도 해제 후 SQLAlchemy·Playwright 버전별 API 검증 완료.

---

*본 문서는 Playwright MCP 검색 → 스펙 분석 → context7/claude-api 검증 순서로 작성됨.*
