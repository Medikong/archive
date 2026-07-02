# DropMong 이벤트 계약

작성일: 2026-07-02

이 문서는 Kafka topic, event envelope, producer/consumer 책임, DLQ와 replay 규칙을 정의한다.

## 1. 이벤트 원칙

- DB 변경과 event 생성은 같은 transaction에 기록한다.
- Kafka publish는 outbox relay가 수행한다.
- consumer는 event id 기준으로 idempotent 해야 한다.
- event payload는 producer가 소유한다.
- consumer가 필요한 정보가 payload에 없으면 API 조회 또는 projection을 명시한다.

## 2. Topic 이름 규칙

| Topic | Producer | Consumer |
| --- | --- | --- |
| `catalog.drop.events.v1` | `catalog-service` | `order-service`, `notification-service` |
| `order.events.v1` | `order-service` | `payment-service` 선택, `notification-service`, analytics |
| `payment.events.v1` | `payment-service` | `order-service`, `notification-service` |
| `notification.events.v1` | `notification-service` | analytics 선택 |
| `dead-letter.events.v1` | platform 또는 consumer | replay tooling |

## 3. 이벤트 envelope

```json
{
  "event_id": "uuid",
  "event_type": "order.created",
  "event_version": 1,
  "occurred_at": "2026-07-02T05:00:00Z",
  "producer": "order-service",
  "aggregate_type": "order",
  "aggregate_id": "order_uuid",
  "correlation_id": "req_123",
  "causation_id": "idem_456",
  "traceparent": "00-...",
  "payload": {}
}
```

필수 필드:

- `event_id`
- `event_type`
- `event_version`
- `occurred_at`
- `producer`
- `aggregate_type`
- `aggregate_id`
- `payload`

## 4. 주문 이벤트

### `order.created`

페이로드:

```json
{
  "order_id": "order_uuid",
  "drop_id": "drop_uuid",
  "customer_id": "user_uuid",
  "quantity": 1,
  "status": "PENDING_PAYMENT",
  "expires_at": "2026-07-02T05:10:00Z"
}
```

소비자:

- `notification-service`: 선택적 주문 접수 알림
- analytics: 주문 funnel

### `order.confirmed`

페이로드:

```json
{
  "order_id": "order_uuid",
  "drop_id": "drop_uuid",
  "customer_id": "user_uuid",
  "payment_id": "payment_uuid",
  "confirmed_at": "2026-07-02T05:02:00Z"
}
```

소비자:

- `notification-service`
- sold-out hint용 catalog projection 선택

### `order.reservation.expired`

페이로드:

```json
{
  "order_id": "order_uuid",
  "drop_id": "drop_uuid",
  "customer_id": "user_uuid",
  "expired_at": "2026-07-02T05:10:01Z"
}
```

소비자:

- `notification-service`

## 5. 결제 이벤트

### `payment.approved`

페이로드:

```json
{
  "payment_id": "payment_uuid",
  "order_id": "order_uuid",
  "customer_id": "user_uuid",
  "amount": 79000,
  "currency": "KRW",
  "approved_at": "2026-07-02T05:01:00Z"
}
```

소비자:

- `order-service`: 가능한 상태이면 order confirm

### `payment.failed`

페이로드:

```json
{
  "payment_id": "payment_uuid",
  "order_id": "order_uuid",
  "customer_id": "user_uuid",
  "reason": "MOCK_FAILURE",
  "failed_at": "2026-07-02T05:01:00Z"
}
```

소비자:

- `order-service`: 가능한 상태이면 order cancel과 reservation release

## 6. 카탈로그 이벤트

### `catalog.drop.updated`

페이로드:

```json
{
  "drop_id": "drop_uuid",
  "product_id": "product_uuid",
  "status": "SCHEDULED",
  "open_at": "2026-07-10T10:00:00Z",
  "cache_tags": ["catalog", "drop:drop_uuid", "product:product_uuid"]
}
```

소비자:

- cache purge 또는 projection builder
- inventory bucket setup이 event-driven일 때만 `order-service`

## 7. Consumer 멱등성

각 consumer는 다음 정보를 유지해야 한다.

```text
consumer_name
event_id
processed_at
handler_version
result
```

처리 규칙:

1. transaction을 시작한다.
2. processed marker table에 `(consumer_name, event_id)`를 insert한다.
3. duplicate이면 성공으로 보고 중단한다.
4. business change를 적용한다.
5. commit한다.

## 8. 재시도와 DLQ

| 실패 | 처리 |
| --- | --- |
| 일시적 DB error | backoff를 적용해 retry |
| schema validation error | DLQ로 전송 |
| 불가능한 상태 전환 | metric 기록, stale event이면 processed 처리 |
| downstream notification provider error | retry 후 DLQ |
| poison event | DLQ 격리와 alert |

DLQ event에는 원본 envelope, consumer 이름, error class, error message, 최초 실패 시각, 마지막 실패 시각, attempt count가 포함되어야 한다.

## 9. 관측성

메트릭:

- `outbox_pending_count`
- `outbox_publish_attempts_total`
- `outbox_publish_failures_total`
- `kafka_consumer_lag`
- `consumer_processed_total`
- `consumer_duplicate_total`
- `consumer_dlq_total`
- `event_handler_duration_seconds`

로그:

- event id
- event type
- aggregate id
- consumer 이름
- correlation id
- trace id
