# ADR 006 - Agent Circuit Breaker & Bulkhead

## 상태

Accepted — 단계 6a·6b·6c 구현·배포 완료 (2026-05). 6d(degradation)는 선택으로 보류.

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
implementation 'io.github.resilience4j:resilience4j-circuitbreaker:2.3.0'
implementation 'io.github.resilience4j:resilience4j-bulkhead:2.3.0'
implementation 'io.github.resilience4j:resilience4j-micrometer:2.3.0'
```

(spring-cloud starter / `resilience4j-spring-boot3` 는 `@CircuitBreaker` 류 AOP
annotation 을 classpath 에 끌어와 의도치 않은 자동 적용 위험이 있어 쓰지 않는다.
core 3 모듈만 도입해 fluent API 로 Bean 을 직접 구성하면, 의존성 추가만으로 기존
spring-web bean 에 부수효과가 생기지 않는다 — 실제 구현은 이 방식을 택했다.)

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

**failure 분류 진리표:**

`AgentHttpRetryer.executeAndParse` 가 반환/throw 한 결과를 다음 표로 판정한다.
조건은 위에서 아래로 평가하며 첫 매치를 적용한다 (cascading).

| 우선순위 | `errorCode` / 예외 | `httpStatus` | `retryable` | 판정 |
|:---:|---|:---:|:---:|---|
| 1 | `AGENT_NETWORK_ERROR` | n/a | any | **FAILURE** |
| 2 | any `AgentInvocationException` | 5xx (500/502/503/504) | any | **FAILURE** |
| 3 | any `AgentInvocationException` | 429 | any | **FAILURE** |
| 4 | any `AgentInvocationException` | 4xx (400/401/403/404/409/422 등) | `false` | **IGNORE** (사용자 입력 오류 — 에이전트 건강 신호 아님) |
| 5 | any `AgentInvocationException` | 4xx | `true` | **FAILURE** (드물지만 에이전트가 일시 장애를 4xx 로 표현한 경우) |
| 6 | 성공 응답 (예외 없음) | n/a | n/a | **SUCCESS** |

이 분류는 Resilience4j 의 `recordExceptions` / `ignoreExceptions` 가 아닌
`recordResult` Predicate 로 구현 — `AgentInvocationException.errorCode` 와
`getHttpStatus()` 까지 본 후 결정해야 하기 때문.

slowCall 은 별도 축으로, `slowCallDurationThreshold` 를 초과한 호출은 성공/실패와
무관하게 slowCall 로 카운트된다.

### 데코레이터 적용 순서

```
요청 → CircuitBreaker → Bulkhead → AgentHttpRetryer.executeAndParse
                                          └─ envelope retry (envelope.retryable + retry_after)
```

**CB 가 최외곽, retryer 가 최내부.** 의미:

- retryer 가 모든 envelope retry 를 소진한 **최종 결과** 를 CB 가 판정한다 → 일시 5xx 가 retry 로 회복되면 CB 에서는 SUCCESS 로 보여 false-open 위험 감소
- Bulkhead 는 retryer 의 retry 시도 전체를 1 slot 으로 묶는다 — retry 중 다른 호출이 끼어들지 않음
- CB 가 OPEN 이면 retryer 를 시도조차 하지 않고 즉시 `AGENT_CIRCUIT_OPEN` 으로 단락 차단

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

micrometer-resilience4j(`TaggedCircuitBreakerMetrics` / `TaggedBulkheadMetrics`)가 자동
binder 로 노출하는 **실제** metric 은 다음과 같다. label 은 `agent` 가 아니라 `name`(= agent
식별자)이다:

- `resilience4j_circuitbreaker_state` — gauge × {name, state}. 현재 상태 series 만 1 (state=closed/open/half_open/...)
- `resilience4j_circuitbreaker_failure_rate` / `_slow_call_rate` — gauge × {name}. 최소 호출수 미만이면 -1
- `resilience4j_circuitbreaker_not_permitted_calls_total` — counter × {name}. OPEN 으로 단락 차단된 호출 수(차단 거부 관측용)
- `resilience4j_bulkhead_available_concurrent_calls` / `_max_allowed_concurrent_calls` — gauge × {name}

본 ADR 초안이 가정한 `agent.circuit_breaker.transition` / `agent.bulkhead.full` 같은 별도
counter 는 실재하지 않는다. 상태 전이는 `resilience4j_circuitbreaker_state` 의 변화로,
bulkhead 포화는 `available_concurrent_calls == 0` 으로 관측한다. Bean 으로
CircuitBreakerRegistry / BulkheadRegistry 를 노출하면 actuator `/actuator/prometheus` 가 가져가며 신규 코드는 없다.

### 5) 단계별 적용

| 단계 | 산출물 | 검증 기준 |
|---|---|---|
| **6a** | Resilience4j 의존성 + `AgentResilienceConfig` (CB/Bulkhead Bean 정의, agent 별 인스턴스 생성). `AgentHttpRetryer` 미수정 — Bean 만 노출. CB/Bulkhead metric 자동 노출 확인 | (1) actuator `/actuator/prometheus` 에 신규 `resilience4j_*` metric 노출. (2) 기존 호출 동작 변화 0 — `grep -rn "@CircuitBreaker\|@Bulkhead\|@Retry\|@TimeLimiter" src/main` 으로 기존 코드에 Resilience4j AOP annotation 부재 확인 (의존성 도입만으로 AOP 가 spring-web bean 에 자동 적용되지 않도록). (3) integration test: 같은 endpoint 부하 1000 req 에 대해 6a 도입 전후 latency p99 차이 ±5% 이내 |
| **6b** | `AgentHttpRetryer.executeAndParse` 가 CB.decorate + Bulkhead.decorate 로 감싸도록 변경. failure 분류 Predicate 도 구현. envelope.retryable=false + 4xx 는 ignore. Open / BulkheadFull 의 envelope 변환 | 4xx 만 발생하는 부하 패턴이 CB 를 열지 않음, 5xx 50% 부하가 CB 를 30 초 OPEN 시킴 (integration test) |
| **6c** | 운영 적용. 알람 룰: `resilience4j_circuitbreaker_state` 가 open/half-open 으로 2분 지속 → warning, 15분 → critical(severity 라벨로 alertmanager Slack/메일 라우팅). Grafana CB/Bulkhead 패널. 알람·대시보드는 중앙 관측 레포(mac-infra)에 위치. **agent 별 fine-tune 은 관측 데이터 확보 후로 보류.** | 7 일간 false-open 0, real outage 시 CB 가 자동 회복 (manual restart 없이 HALF_OPEN → CLOSED 복귀) |
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
- **저트래픽 시 장애 감지 지연**: `COUNT_BASED` slidingWindow(20) 는 호출 빈도가 낮으면 window 가 수십 분에 걸칠 수 있어, 오래된 성공이 window 에 남아 실제 장애가 늦게 OPEN 으로 반영될 수 있다. 운영 단계 6c 에서 agent 별 호출 빈도를 보고 `slidingWindowSize` 또는 `slidingWindowType=TIME_BASED` 로 재조정
- **Bulkhead 10 의 균등 분포 가정**: `Tomcat 200 / 10 agent` 계산은 호출 빈도가 균등하다는 가정이다. 실제로는 `draft_agent` / `style_agent` 가 더 자주 호출되므로 false rejection 위험 — 단계 6c 에서 호출 빈도 기반 재조정
- **단계 6b 의 failure Predicate 가 ADR 005 envelope 에 강결합** — 의존하는 envelope 필드:
  - `AgentInvocationException.getErrorCode()` ← `AGENT_NETWORK_ERROR` 분기 (진리표 우선순위 1)
  - `AgentInvocationException.getHttpStatus()` ← 5xx/429/4xx 분기 (진리표 우선순위 2-5)
  - `AgentInvocationException.isRetryable()` ← 4xx 의 IGNORE/FAILURE 갈림길 (진리표 우선순위 4-5)

  위 3 시그니처 중 하나라도 변경되면 failure Predicate 동반 갱신 필수. ADR 005 본문에도 cross-reference 추가 권장 (별도 작업)
- 분산 환경 (orchestrator instance 다수) 에서 CB 는 instance-local — 각 인스턴스가 독립 학습. 운영상 큰 문제 아니나, 한 인스턴스가 학습한 OPEN 상태가 다른 인스턴스에 즉시 전파되지는 않음. 단계 6c 의 알람 룰은 instance 별 metric 으로 분리해서 봐야 함

**중립**
- micrometer 자동 노출 `resilience4j_*` metric → Grafana 패널 / 알람 룰 신규 작성 필요. ADR 005 단계 2e 의 패턴 (PR #33) 을 그대로 차용

**대안 (rejected)**
- **Hystrix**: 2019 maintenance mode. 신규 도입 부적합
- **순수 RetryTemplate + 수동 gate**: CB 의 자동 회복·HALF_OPEN trial 같은 핵심 동작을 직접 구현해야 함 — 재발명
- **service mesh (Istio outlier detection)**: 인프라 비용·복잡도 대비 효익 낮음. 현 환경은 단일 docker-compose
- **글로벌 1 CB / 1 Bulkhead**: agent 격리 효익 손실

**관련 ADR**
- ADR 005: 에이전트 에러 envelope. 이 ADR 의 failure 분류가 envelope 의
  `retryable` + `httpStatus` 에 의존
- ADR 003: ID strategy. 신규 metric 의 trace_id label 이 사용
