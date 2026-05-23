# ADR 005 - Agent Error Response Format

## 상태

승인됨

## 날짜

2026-05-23

## 맥락

오케스트레이터는 10개의 FastAPI 에이전트(`voice_profile`, `style`, `draft`, `review`,
`outline`, `meta`, `hero_photo`, `photo_grouping`, `privacy_safety`, `quality_score`)를
HTTP 로 호출한다. 현재 에러 처리는 다음과 같이 일관성이 없다.

**에이전트 측 현황**

- `voice_profile_agent` 만 `HTTPException` 으로 RFC HTTP status + `detail` 반환.
  `detail` 의 언어가 한국어 / 영어 혼용 (`"인증이 필요합니다."` vs `"voice profile not found"`)
- 나머지 9개 에이전트는 **HTTP 200 + 본문 `status: "error"` 패턴** 으로 에러를 표현.
  성공/실패가 같은 status code 라 retry 판단·관측에 약함
- FastAPI 의 `RequestValidationError` (422) 는 어느 에이전트에서도 wrap 되지 않아
  default `{detail: [{loc, msg, type}, ...]}` 본문이 그대로 흘러나간다

**orchestrator 측 현황**

- 클라이언트마다 catch 처리가 3가지 패턴으로 흩어져 있음
  (단순 status+body / response body truncate / silent fallback)
- 모두 `IllegalStateException` 으로 전달 — `error_code` 없음, 사용자-facing 메시지
  분리 없음, raw response body 가 그대로 노출될 위험
- `AgentHttpRetryer` 는 HTTP status 5xx / 429 만 retry. 에이전트가 "이 4xx 는 사실
  retryable" 같은 의도를 표현할 수단 없음
- `WorkflowController` 에 `@ExceptionHandler` / `@ControllerAdvice` 가 아직 없다.
  현재 `executeRestyle` catch 블록은 `IllegalStateException.getMessage()` 를 그대로
  `workflow.markFailed()` 로 흘려보내 raw response body 가 사용자 화면에 노출된다

**운영·UX 측면 영향**

- 사용자 화면에 raw stack trace 일부가 노출 (콘솔 새 글쓰기 / 작업 기록 실패 화면)
- 같은 종류의 장애가 에이전트마다 다른 모양으로 보여 root cause 추적이 어려움
- retry 정책이 status 추정에 의존해 의도와 불일치할 수 있음
- request ↔ error 상관관계를 추적할 공통 식별자가 없어 멀티-에이전트 한 워크플로의
  실패 한 건을 따라가려면 timestamp + agent 이름으로 수작업 매칭해야 한다

## 결정

다음을 모든 에이전트와 orchestrator 클라이언트의 **표준 에러 응답 형식** 으로 한다.

### 에이전트 에러 응답 스키마

```json
{
  "error_code": "VOICE_PROFILE_NOT_FOUND",
  "message": "voice profile not found",
  "user_message": "말투 프로필을 찾을 수 없습니다.",
  "retryable": false,
  "retry_after_seconds": null,
  "trace_id": "wf_090d8cb4_step_meta_2026-05-23T14:03:52Z",
  "details": null
}
```

| 필드 | 타입 | 필수 | 의미 |
|------|------|:---:|------|
| `error_code` | `string` (UPPER_SNAKE_CASE) | 예 | 안정적 식별자. agent prefix + 도메인 명시 권장 (`VOICE_PROFILE_NOT_FOUND`, `STYLE_LLM_TIMEOUT` 등) |
| `message` | `string` (영어) | 예 | 디버그·로그용 자세한 설명 |
| `user_message` | `string` (한국어) | 예 | 콘솔에 그대로 표시 가능한 사용자-facing 메시지. 내부 경로·스택·variable 노출 금지 |
| `retryable` | `boolean` | 예 | 클라이언트가 같은 요청을 그대로 다시 보내도 다른 결과가 나올 수 있는지에 대한 에이전트의 의도 |
| `retry_after_seconds` | `integer | null` | 아니오 | rate limit / cold-start 등 에이전트가 backoff 윈도를 알 때만 채운다. orchestrator 는 명시 시 이 값을 우선 사용 |
| `trace_id` | `string | null` | 아니오 | request ↔ log ↔ error 상관 식별자. orchestrator 는 `X-Request-Id` 헤더로 trace_id 를 내려보내고, 에이전트는 받은 그대로 echo. 한 워크플로의 여러 에이전트 호출을 묶는 키 |
| `details` | `array | null` | 아니오 | 필드 단위 validation 실패 같은 구조화된 sub-errors. 각 원소는 `{loc: string[], msg: string, type: string}` 형식 (FastAPI `RequestValidationError` 와 호환) |

### HTTP 규칙

- HTTP status code 는 **RFC 표준 의미 그대로** 사용한다 (400 / 401 / 403 / 404 / 409 /
  422 / 429 / 500 / 502 / 503 / 504).
- 기존 "HTTP 200 + `status:"error"`" 패턴은 **deprecated**. 점진 제거한다.
- 정상 응답에는 `error_code` 필드가 없어야 한다 (구분 가능).
- FastAPI 의 `RequestValidationError` 는 기본 422 본문이 표준 envelope 과 다르므로
  각 에이전트에 `@app.exception_handler(RequestValidationError)` 를 두어 표준
  스키마(`error_code: "VALIDATION_FAILED"`, `details: exc.errors()`) 로 wrap 한다.
  `HTTPException` 도 동일하게 wrap 한다.

### orchestrator 측 매핑

- 새 타입 `AgentInvocationException(errorCode, userMessage, retryable,
  retryAfterSeconds, httpStatus, traceId, details, cause)` 를 도입하고 모든 클라이언트
  가 기존 `IllegalStateException` 대신 이걸 던진다.
- **`@ControllerAdvice` / `@ExceptionHandler` 를 신설한다.** 현재 `WorkflowController`
  에는 exception handler 자체가 없고 `executeRestyle` 의 catch 가
  `IllegalStateException.getMessage()` 를 그대로 `workflow.markFailed()` 로 흘려보낸
  다 — 이를 새 advice 가 `AgentInvocationException` 을 잡아 `user_message` 만 응답
  본문으로, 나머지(`message` / 스택 / raw body) 는 로그·`last_error_message` 에만
  남기도록 바꾼다.
- `AgentHttpRetryer` 는 `retryable=true` 면 status 와 무관하게 retry, `retry_after_seconds`
  가 있으면 그 값을 backoff 로 사용한다. `retryable` 이 명시되지 않으면 기존 동작
  (5xx / 429 retry + config backoff) 으로 fallback.
- 모든 클라이언트는 outbound 호출 시 `X-Request-Id` 헤더에 `trace_id` 를 동봉하고,
  응답의 `trace_id` 를 그대로 받아 `AgentInvocationException` 에 보존한다.

### 점진 적용 정책

| 단계 | 범위 |
|------|------|
| 1 | ADR + 표준 spec 문서화 (이 ADR — 본 PR) |
| 2a | orchestrator 인프라 신설 — `AgentInvocationException` / `@ControllerAdvice` / 표준 파서 유틸. 기존 client 는 아직 `IllegalStateException` 던지지만 advice 가 둘 다 처리 (legacy fallback) |
| 2b | reference 에이전트 적용 — `voice_profile_agent` (이미 `HTTPException` 사용 중이라 변경량이 적다) |
| 2c | reference 클라이언트 적용 — `StyleAgentClient` (정통 `IllegalStateException` → `AgentInvocationException` 전환 시연용). `VoiceProfileAgentClient` 는 silent-fallback 패턴이라 정통 reference 로 부족하므로 함께 묶는다 |
| 3 | 나머지 8개 에이전트와 클라이언트 mechanical migration. 한 에이전트씩 별도 PR |
| 4 | "HTTP 200 + `status:"error"`" 패턴 제거 및 deprecated 경로 정리. orchestrator advice 의 legacy fallback 도 함께 제거 |

각 단계가 끝나면 `docs/dev-log.md` 와 `docs/next-session-handoff.md` 를 갱신한다.

## 이유

- **관측성**: HTTP status code 정상 사용으로 모니터링·alert 룰이 자연스러워진다.
- **사용자 UX**: `user_message` 가 항상 있어 콘솔에 raw stack 이 노출되는 사고를
  스키마 차원에서 차단한다.
- **재시도 정책의 명시성**: `retryable` 을 에이전트가 표명하면 의도가 코드 밖으로
  드러난다. `AgentHttpRetryer` 의 status 휴리스틱은 fallback 으로만 유지된다.
- **국제화 부담 분산**: `user_message` 를 에이전트가 만들면 컨텍스트가 가장 가까운
  위치에서 메시지가 생성된다. (대안: orchestrator 가 `error_code` → 메시지 매핑을
  중앙 보유. 검토했으나 도메인 지식 중복·번역 회귀가 잦아 보류)
- **`details` 분리**: validation 실패는 단일 문자열로 환원되면 어느 필드인지 잃는다.
  FastAPI 의 `RequestValidationError.errors()` 형태(`{loc, msg, type}`) 를 그대로
  보존하면 콘솔이 필드 단위로 강조 표시 가능.
- **`retry_after_seconds`**: 429 / cold-start LLM 등 에이전트가 backoff 윈도를 더
  잘 아는 경우가 있다. orchestrator 의 고정 `backoffMillis` 가 의도와 어긋나는 사고를
  방지.
- **`trace_id`**: 한 워크플로가 10개 가까운 에이전트를 부른다. 같은 워크플로의
  단계별 실패를 timestamp 매칭 없이 빠르게 묶을 수 있어야 한다. `X-Request-Id` 헤더
  echo 패턴은 OpenTelemetry / RFC 9457 사례와도 일치.

## 결과

- 새 ADR 005 가 등록되어, 이후 에이전트 변경 PR 은 이 표준을 기준으로 리뷰한다.
- 후속 PR 시리즈: 2a(orchestrator 인프라 신설) → 2b(voice_profile_agent reference)
  → 2c(StyleAgentClient 정통 reference) → 단계 3 mechanical migration 8개 → 단계 4
  legacy 제거.
- `IllegalStateException` 에 의존하던 기존 client 와 테스트는 단계 2c·3 에서 함께
  수정. 테스트는 `AgentInvocationException` assertion 으로 갱신.
- 단계 4 이전까지는 두 패턴이 공존하므로, orchestrator 파서는 표준·legacy(`HTTP 200
  + status:"error"`, `IllegalStateException` 메시지) 모두 처리할 수 있어야 한다.
- 새 `@ControllerAdvice` 는 `AgentInvocationException` 외에도 일반 `Exception`
  fallback 도 가져야 한다 (마이그레이션 중 미전환 에이전트의 raw 응답 노출 방지).
