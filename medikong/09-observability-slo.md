# DropMong 관측성과 SLO

작성일: 2026-07-02

이 문서는 DropMong의 SLI, SLO, dashboard, alert, trace/log 기준을 정의한다.

## 1. 관측 목표

- oversell 여부를 즉시 알 수 있어야 한다.
- admitted traffic과 rejected/queued traffic을 분리해서 봐야 한다.
- 주문, 결제, 이벤트, 알림을 correlation id로 추적할 수 있어야 한다.
- Kafka lag, outbox lag, DLQ 증가를 배포와 연결해서 볼 수 있어야 한다.
- canary 승격과 rollback 판단이 Prometheus query로 가능해야 한다.

## 2. SLI와 SLO

| 영역 | SLI | 초기 목표 |
| --- | --- | --- |
| 재고 정합성 | `oversell_count` | 항상 `0` |
| Checkout 가용성 | admitted `POST /orders` success rate | injected failure 외에는 `>= 99%` |
| Checkout latency | admitted `POST /orders` p95 | local target `< 500ms`, 환경별 조정 |
| Payment event 처리 | payment event processing success rate | `>= 99.5%` |
| Outbox | pending outbox age p95 | `< 30s` |
| Kafka | consumer lag 해소 시간 | spike 이후 `< 5m` |
| Notification 격리 | notification outage가 order error rate에 주는 영향 | 측정 가능한 증가 없음 |
| Rollback | SLO 위반 후 rollback 시간 | 감지 후 `< 2m` |

## 3. 필수 메트릭

### API

- `http_requests_total{service,route,method,status,admission_result}`
- `http_request_duration_seconds{service,route,method,admission_result}`
- `admission_requests_total{result}`
- `admission_queue_time_seconds`

### 주문

- `orders_created_total`
- `orders_confirmed_total`
- `orders_cancelled_total`
- `reservations_active`
- `reservations_expired_total`
- `inventory_reserved_quantity`
- `inventory_confirmed_quantity`
- `inventory_total_quantity`
- `oversell_count`
- `idempotency_replay_total`
- `idempotency_conflict_total`

### 결제

- `payments_requested_total`
- `payments_approved_total`
- `payments_failed_total`
- `payments_delayed_total`
- `payment_event_handler_failures_total`

### 이벤트

- `outbox_pending_count`
- `outbox_oldest_pending_age_seconds`
- `outbox_publish_failures_total`
- `kafka_consumer_lag`
- `consumer_dlq_total`
- `consumer_duplicate_total`

### 배포

- `rollout_phase`
- `canary_weight`
- `canary_error_rate`
- `rollback_total`

## 4. Label 정책

높은 cardinality label 허용 기준:

| Label | 허용 여부 | 이유 |
| --- | --- | --- |
| `service` | yes | bounded |
| `route` | yes | templated route만 허용 |
| `method` | yes | bounded |
| `status` | yes | bounded |
| `admission_result` | yes | admitted, rejected, queued |
| `version` | yes | rollout analysis |
| `drop_id` | limited | cardinality가 통제된 dedicated business metric에서만 사용 |
| `order_id` | no | high cardinality |
| `customer_id` | no | high cardinality이고 민감 정보 |
| `payment_id` | no | high cardinality |

동적 ID는 metric label이 아니라 log와 trace에 넣는다.

## 5. Logging

모든 structured log는 다음을 포함해야 한다.

- `timestamp`
- `level`
- `service`
- `event`
- `request_id`
- `trace_id`
- 사용 가능한 경우 `span_id`
- `correlation_id`
- 필요한 경우와 정책이 허용하는 경우에만 `user_id`
- `order_id` 같은 business id는 log에만 기록

기록하지 않을 항목:

- password
- JWT token
- payment secret
- 전체 authorization header

## 6. Tracing

Trace 전파:

```text
frontend 또는 gateway
-> Istio
-> service HTTP span
-> DB span
-> Kafka produce span
-> Kafka consume span
-> downstream service span
```

필수 trace attribute:

- `service.name`
- `http.route`
- `messaging.system`
- `messaging.destination.name`
- `db.system`
- 안전한 경우 `dropmong.order_id`
- 안전한 경우 `dropmong.drop_id`

## 7. Dashboard

### 드롭 오픈 개요

패널:

- admitted, rejected, queued RPS
- accepted order p95/p99
- status별 order success/failure
- active reservation
- confirmed order
- oversell count
- DB transaction latency

### 이벤트 상태

패널:

- outbox pending count
- 가장 오래된 outbox age
- publish failure
- consumer group별 Kafka lag
- DLQ depth
- duplicate event count

### 릴리즈 상태

패널:

- canary weight
- stable 대비 canary error rate
- stable 대비 canary p95/p99
- rollback count
- deployment event

### 알림 격리

패널:

- notification consumer lag
- notification DLQ
- order error rate overlay
- event replay result

## 8. Alert

| Alert | 심각도 | 조건 |
| --- | --- | --- |
| `OversellDetected` | critical | `oversell_count > 0` |
| `CheckoutErrorRateHigh` | page | admitted order error rate가 threshold 초과 |
| `CheckoutLatencyHigh` | page | admitted p95가 threshold 초과 |
| `OutboxStuck` | page | oldest pending age가 threshold 초과 |
| `KafkaLagNotDraining` | ticket/page | 지속 시간 동안 lag 증가 |
| `DLQIncreasing` | ticket/page | DLQ 증가 |
| `CanarySLOBreach` | page | rollout analysis 실패 |
| `AdmissionRejectSpike` | ticket | 예상되지 않은 rejected RPS spike |

## 9. 증거 경로

권장 증거 위치:

```text
workspaces/docs/evidence/dropmong/
  loadtest/
  observability/
  release/
  incidents/
```

각 증거 항목에는 다음을 포함한다.

- 날짜와 환경
- 관련 repo의 git SHA
- 사용한 command
- dashboard screenshot 또는 query output
- 결과 요약
- follow-up issue
