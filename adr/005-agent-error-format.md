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

**orchestrator 측 현황**

- 클라이언트마다 catch 처리가 3가지 패턴으로 흩어져 있음
  (단순 status+body / response body truncate / silent fallback)
- 모두 `IllegalStateException` 으로 전달 — `error_code` 없음, 사용자-facing 메시지
  분리 없음, raw response body 가 그대로 노출될 위험
- `AgentHttpRetryer` 는 HTTP status 5xx / 429 만 retry. 에이전트가 "이 4xx 는 사실
  retryable" 같은 의도를 표현할 수단 없음

운영·UX 측면 영향

- 사용자 화면에 raw stack trace 일부가 노출 (콘솔 새 글쓰기 / 작업 기록 실패 화면)
- 같은 종류의 장애가 에이전트마다 다른 모양으로 보여 root cause 추적이 어려움
- retry 정책이 status 추정에 의존해 의도와 불일치할 수 있음

## 결정

다음을 모든 에이전트와 orchestrator 클라이언트의 **표준 에러 응답 형식** 으로 한다.

### 에이전트 에러 응답 스키마

```json
{
  "error_code": "VOICE_PROFILE_NOT_FOUND",
  "message": "voice profile not found",
  "user_message": "말투 프로필을 찾을 수 없습니다.",
  "retryable": false
}
```

| 필드 | 타입 | 의미 |
|------|------|------|
| `error_code` | `string` (UPPER_SNAKE_CASE) | 안정적 식별자. agent prefix + 도메인 명시 권장 (`VOICE_PROFILE_NOT_FOUND`, `STYLE_LLM_TIMEOUT` 등) |
| `message` | `string` (영어) | 디버그·로그용 자세한 설명 |
| `user_message` | `string` (한국어) | 콘솔에 그대로 표시 가능한 사용자-facing 메시지. 내부 경로·스택·variable 노출 금지 |
| `retryable` | `boolean` | 클라이언트가 같은 요청을 그대로 다시 보내도 다른 결과가 나올 수 있는지에 대한 에이전트의 의도 |

### HTTP 규칙

- HTTP status code 는 **RFC 표준 의미 그대로** 사용한다 (400 / 401 / 403 / 404 / 409 /
  422 / 429 / 500 / 502 / 503 / 504).
- 기존 "HTTP 200 + `status:"error"`" 패턴은 **deprecated**. 점진 제거한다.
- 정상 응답에는 `error_code` 필드가 없어야 한다 (구분 가능).

### orchestrator 측 매핑

- 새 타입 `AgentInvocationException(errorCode, userMessage, retryable, httpStatus, cause)`
  를 도입하고 모든 클라이언트가 기존 `IllegalStateException` 대신 이걸 던진다.
- `WorkflowController` 의 exception handler 는 `user_message` 만 응답 본문으로 노출.
  `message` / 스택 / 본문 raw 는 로그·`last_error_message` 에만 남긴다.
- `AgentHttpRetryer` 는 `retryable=true` 면 status 와 무관하게 retry. `retryable` 이
  명시되지 않으면 기존 동작(5xx / 429 retry) 으로 fallback.

### 점진 적용 정책

| 단계 | 범위 |
|------|------|
| 1 | ADR + 표준 spec 문서화 (이 ADR — 본 PR) |
| 2 | reference 구현 = `voice_profile_agent` (이미 `HTTPException` 사용 중) + orchestrator `VoiceProfileAgentClient` 표준 파서 |
| 3 | 나머지 9개 에이전트와 클라이언트 mechanical migration. 한 에이전트씩 별도 PR |
| 4 | "HTTP 200 + `status:"error"`" 패턴 제거 및 deprecated 경로 정리 |

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

## 결과

- 새 ADR 005 가 등록되어, 이후 에이전트 변경 PR 은 이 표준을 기준으로 리뷰한다.
- 후속 PR 시리즈: voice_profile_agent reference → mechanical migration 9개.
- `IllegalStateException` 에 의존하던 기존 client 와 테스트는 단계 3 에서 함께 수정.
- 단계 4 이전까지는 두 패턴이 공존하므로, orchestrator 파서는 표준·legacy 모두
  처리할 수 있어야 한다.
