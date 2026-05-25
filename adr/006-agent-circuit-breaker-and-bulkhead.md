# ADR 006 - Agent Circuit Breaker & Bulkhead

## 상태

초안 (Proposed)

## 날짜

2026-05-25

## 맥락

ADR 005 가 완료되면서 orchestrator → 10 FastAPI 에이전트 호출의 에러 응답이
표준화되고 (`AgentInvocationException` envelope), retry 정책이 에이전트의
`retryable` / `retry_after_seconds` 신호를 1차 사용하도록 정착됐다. 동시에 단계 2e
(PR #33) 의 `agent.invocation.error` Prometheus Counter 로 agent / error_code /
status 차원별 에러율을 관측할 수 있다.

그러나 다음 두 가지는 여전히 미해결이다.

**1. cascade failure 위험**

`AgentHttpRetryer.executeAndParse` 는 한 에이전트가 응답 안 함 / 매우 느림 상태로
들어가도 다음 같은 패턴으로 spring-web 의 톰캣 worker thread 를 묶는다:

- 단일 호출이 RestClient 의 read-timeout (현재 60초 가까이) 까지 hold
- 에이전트가 envelope `retryable=true` 로 자체 신호를 보내면 retryer 가
  `maxAttempts` 까지 추가 시도
- 그 사이 같은 에이전트로 향하는 신규 워크플로 요청도 같은 운명

특히 LLM 호출 (Ollama 기반 `style_agent` / `draft_agent` / `review_agent` /
`meta_agent`) 은 cold-start 시 1 호출이 30 ~ 60 초를 잡아먹기 쉽다. 그 동안
unrelated 워크플로의 진행이 막힌다 — 한 에이전트의 부분 장애가 시스템 전체의
응답성 저하로 전파 (cascade failure).

`AgentHttpRetryer.executeAndParse` 가 envelope.retryable 을 honor 하면서
backoff 계산까지 끝내고 throw 하는 구조라, 다음 호출이 "이 에이전트는 지금 아픔"
이라는 사실을 알 방법이 없다 — 매번 같은 retry 비용을 처음부터 다시 치른다.

**2. 부분 장애 시 우아한 다운그레이드 부재**

ADR 005 단계 2e 의 metric 으로 "오늘 quality_score_agent 가 50% 실패 중" 같은
관측은 가능하지만, orchestrator 가 이 정보를 사용하지 않는다. 운영자가 대시보드를
보고 수동 개입 (replicas 늘리기 / 에이전트 재시작) 해야 다음 사용자 요청이 살아난다.

ADR 005 §결과 의 "단계 5: 별도 ADR (circuit breaker / bulkhead)" 약속이 이 ADR 의
범위다.

## 결정

**Resilience4j 기반의 per-agent Circuit Breaker + Bulkhead 도입.**

각 에이전트(예: `style`, `draft`, `meta` …) 에 독립된 CB 인스턴스와 bulkhead 를
두고, `AgentHttpRetryer.executeAndParse` 가 진입점에서 두 primitive 를 통과한 후에만
실제 RestClient 호출을 수행한다.

### 의존성

```
implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j'
```

(Spring Cloud Circuit Breaker abstraction 보다 Resilience4j 의 fluent API 가
세부 튜닝에 유리하므로 starter 만 활용하고 Bean 은 직접 구성한다)

### 1) Circuit Breaker — 에이전트당 1 인스턴스

CLOSED → OPEN → HALF_OPEN 표준 3 상태. 키는 PR #33 의 metric `agent` tag 와 동일한
agent 식별자 (예: `voice-profile`, `style`, `draft`, …) — 10 agent + `unknown`
fallback 으로 bounded.

| 파라미터 | 기본값 | 근거 |
|---|---|---|
| `slidingWindowType` | `COUNT_BASED` | 트래픽이 낮을 수 있어 time-based 는 노이즈 위험 |
| `slidingWindowSize` | 20 | 통계 안정성 vs 반응성 절충 |
| `minimumNumberOfCalls` | 10 | CB 가 너무 일찍 열리지 않도록 |
| `failureRateThreshold` | 50 (%) | 절반 이상 실패하면 명백한 장애 |
| `slowCallDurationThreshold` | 30s | LLM 평균 응답 (10~20초) 보다 큰 값 |
| `slowCallRateThreshold` | 70 (%) | slow call 도 실패로 카운트 |
| `waitDurationInOpenState` | 30s | half-open 전 회복 시간 |
| `permittedNumberOfCallsInHalfOpenState` | 5 | trial 호출 수 |
| `automaticTransitionFromOpenToHalfOpenEnabled` | true | 별도 polling 불필요 |

**failure 정의:**
- `AgentInvocationException` 중 `retryable=false` 또는 `httpStatus ∈ {500, 502, 503, 504, 429}` 이면 failure 로 카운트
- `AgentInvocationException` 중 `retryable=false` 이고 4xx (404, 422 등) 는 **failure 로 카운트하지 않음** (사용자 입력 오류는 에이전트 건강의 신호가 아님)
- Network exception (ResourceAccessException → `AGENT_NETWORK_ERROR`) 는 failure

이 분류는 Resilience4j 의 `recordExceptions` / `ignoreExceptions` 가 아닌
`recordResult` Predicate 로 구현 — `AgentInvocationException.errorCode` 와
`getHttpStatus()` 까지 본 후 결정해야 하기 때문.

### 2) Bulkhead — 에이전트당 semaphore

| 파라미터 | 기본값 | 근거 |
|---|---|---|
| `type` | `SEMAPHORE` | thread-pool bulkhead 는 비동기 wrapper 필요 — 현재 동기 호출과 incompatible |
| `maxConcurrentCalls` | 10 | spring-web Tomcat default thread 200 / 10 에이전트 = 20 thread/agent 절반 |
| `maxWaitDuration` | 0 (즉시 reject) | wait 가 되면 우리가 피하려는 thread hold 가 다시 발생 |

Bulkhead 가 reject 하면 `BulkheadFullException` → orchestrator 가
`AgentInvocationException` 으로 변환:

```
errorCode:     AGENT_BULKHEAD_FULL
retryable:     true
retry_after_seconds: 5  // jittered 1~10
httpStatus:    503
user_message:  에이전트 호출이 일시적으로 많아 잠시 후 다시 시도해 주세요.
```

### 3) Open 상태 응답

CB 가 OPEN 일 때 `executeAndParse` 는 RestClient 호출을 시도하지 않고 즉시:

```
errorCode:     AGENT_CIRCUIT_OPEN
retryable:     true
retry_after_seconds: <CB 의 남은 wait duration, 1초 단위 ceil>
httpStatus:    503
user_message:  에이전트(<agentName>)가 잠시 중단 상태입니다. 잠시 후 다시 시도해 주세요.
```

ADR 005 의 advice 가 이 envelope 을 그대로 사용자 화면 / SSE 에 전달하므로 별도 처리 불필요.

### 4) 관측성

- 신규 metric: `agent.circuit_breaker.state` — gauge 0/1 × {agent, state}. CB 상태 변화 즉시 노출
- 신규 metric: `agent.circuit_breaker.transition` — counter × {agent, from, to}. 누적 transition 회수
- 신규 metric: `agent.bulkhead.available_concurrent_calls` — gauge × {agent}
- 신규 metric: `agent.bulkhead.full` — counter × {agent}. reject 누적

micrometer-resilience4j 가 위 4 종을 자동 binder 로 노출하므로 신규 코드 없음 —
Bean 으로 CircuitBreakerRegistry / BulkheadRegistry 를 노출하면 actuator 가 가져간다.

### 5) 단계별 적용

| 단계 | 산출물 | 검증 기준 |
|---|---|---|
| **6a** | Resilience4j 의존성 + `AgentResilienceConfig` (CB/Bulkhead Bean 정의, agent 별 인스턴스 생성). `AgentHttpRetryer` 미수정 — Bean 만 노출. metric 4 종 자동 노출 확인 | actuator `/actuator/prometheus` 에 신규 metric 4 종 노출. 기존 호출 동작 변화 0 |
| **6b** | `AgentHttpRetryer.executeAndParse` 가 CB.decorate + Bulkhead.decorate 로 감싸도록 변경. failure 분류 Predicate 도 구현. envelope.retryable=false + 4xx 는 ignore. Open / BulkheadFull 의 envelope 변환 | 4xx 만 발생하는 부하 패턴이 CB 를 열지 않음, 5xx 50% 부하가 CB 를 30 초 OPEN 시킴 (integration test) |
| **6c** | 운영 적용. agent 별 fine-tune (예: `meta_agent` 는 slowCall 임계 60s). 알람 룰: `agent.circuit_breaker.transition{to="OPEN"} > 0` 가 Slack 통지 | 7 일간 false-open 0, real outage 시 CB 가 자동 회복 (manual restart 없이 HALF_OPEN → CLOSED 복귀) |
| **6d** (선택) | per-workflow degradation. 예: `meta_agent` 가 OPEN 이면 워크플로를 META_SKIPPED 로 완료시키고 콘솔에 "메타 생성 누락" 표시 | UX QA 통과 |

6a-c 는 필수. 6d 는 도메인 trade-off 가 커서 별도 PR / discussion 으로 분리.

## 이유

**왜 Resilience4j?**

- Spring Cloud Circuit Breaker 의 abstraction 위에 가장 풍부한 implementation
- Micrometer 통합 binder 가 표준 — PR #33 의 `MeterRegistry` 그대로 재사용 가능
- 의도된 동기 호출 패턴 (executeAndParse) 과 호환 — Hystrix 의 thread-pool 강제와 다름
- ADR 005 의 envelope.retryable + httpStatus 를 그대로 failure Predicate 에 입력 가능 — failure 정의가 도메인-특화로 가능 (다른 라이브러리는 Exception class 기준만 받음)

**왜 per-agent (글로벌 1 CB 가 아님)?**

- Ollama 죽음 (`style`, `draft`, `meta`, `review` 영향) 이 `voice_profile_agent`
  (Ollama 비의존) 까지 차단하면 회복 후 워크플로 전체가 unnecessary 영향
- PR #33 의 metric 이 이미 agent 차원으로 분리 — CB 도 같은 차원이 합리적
- agent 수가 10 으로 작아 cardinality 부담 없음

**왜 bulkhead 도 함께?**

- CB 만 있으면 OPEN 직전까지 worker thread 가 슬로우 호출에 묶인다 — bulkhead 가 동시 호출 수 자체를 cap
- Semaphore bulkhead 라 추가 thread pool / context switch 비용 없음

**왜 4xx (`retryable=false`) 를 failure 로 안 세나?**

- 404 / 422 는 사용자 입력 / 데이터 문제 — 에이전트 건강과 무관
- 이런 입력이 폭주하면 CB 가 잘못 열려 정상 워크플로까지 차단됨
- 이 분류는 ADR 005 envelope 의 `retryable=false` + `httpStatus < 500` 으로 깔끔하게 식별 가능

## 결과

**Pros**
- 한 에이전트 장애가 다른 에이전트 호출을 막지 않음 (cascade 차단)
- LLM cold-start 같은 slow-call 도 slow-call-rate 로 OPEN 가능 — 단순 5xx 만 보던 retry 정책의 사각지대 보강
- 자동 회복 (HALF_OPEN trial → CLOSED) 으로 운영자 수동 개입 빈도 감소
- Open 시 사용자에게 `retry_after_seconds` 가 명시된 envelope 응답 — UX 일관성

**Cons / 위험**
- 설정 knob 증가 (CB 8 개 + Bulkhead 3 개 × 10 agent 잠재) — fine-tune 부담
- false-open 위험: minimumNumberOfCalls / failureRateThreshold 설정이 잘못되면 정상 트래픽을 차단
- 단계 6b 의 failure Predicate 가 ADR 005 envelope 에 강결합 — envelope 스펙 변경 시 함께 갱신 필요 (트레일러로 명시)
- 분산 환경 (orchestrator instance 다수) 에서 CB 는 instance-local — 각 인스턴스가 독립 학습. 운영상 큰 문제 아니나, 한 인스턴스가 학습한 OPEN 상태가 다른 인스턴스에 즉시 전파되지는 않음

**중립**
- micrometer 자동 노출 metric 4 종 → Grafana 패널 / 알람 룰 신규 작성 필요. ADR 005 단계 2e 의 패턴 (PR #33) 을 그대로 차용

**대안 (rejected)**
- **Hystrix**: 2019 maintenance mode. 신규 도입 부적합
- **순수 RetryTemplate + 수동 gate**: CB 의 자동 회복·HALF_OPEN trial 같은 핵심 동작을 직접 구현해야 함 — 재발명
- **service mesh (Istio outlier detection)**: 인프라 비용·복잡도 대비 효익 낮음. 현 환경은 단일 docker-compose
- **글로벌 1 CB / 1 Bulkhead**: agent 격리 효익 손실

**관련 ADR**
- ADR 005: 에이전트 에러 envelope. 이 ADR 의 failure 분류가 envelope 의
  `retryable` + `httpStatus` 에 의존
- ADR 003: ID strategy. 신규 metric 의 trace_id label 이 사용
