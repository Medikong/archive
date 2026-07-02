# DropMong 데이터 설계

작성일: 2026-07-02

이 문서는 DropMong의 데이터 소유권, 주요 테이블, 인덱스, 트랜잭션 경계를 정의한다.

## 1. 데이터베이스 소유권

| 서비스 | 저장소 | 주요 데이터 |
| --- | --- | --- |
| `auth-service` | PostgreSQL | users, credentials, roles |
| `catalog-service` | PostgreSQL | products, drops, drop schedules, catalog outbox |
| `order-service` | PostgreSQL | inventory buckets, orders, reservations, idempotency keys, outbox |
| `payment-service` | PostgreSQL | payments, idempotency keys, outbox |
| `notification-service` | PostgreSQL or MongoDB | notifications, processed events |

DB는 서비스별로 논리 분리한다. 1차 로컬 구현에서 같은 PostgreSQL instance를 쓰더라도 schema는 서비스별로 분리한다.

## 2. order-service 스키마

### `inventory_buckets`

| 컬럼 | 타입 | 규칙 |
| --- | --- | --- |
| `drop_id` | uuid | primary key |
| `total_quantity` | integer | `> 0` |
| `reserved_quantity` | integer | `>= 0` |
| `confirmed_quantity` | integer | `>= 0` |
| `version` | bigint | optimistic locking 후보 |
| `created_at` | timestamptz | required |
| `updated_at` | timestamptz | required |

제약 조건:

```sql
CHECK (reserved_quantity >= 0)
CHECK (confirmed_quantity >= 0)
CHECK (reserved_quantity + confirmed_quantity <= total_quantity)
```

### `orders`

| 컬럼 | 타입 | 규칙 |
| --- | --- | --- |
| `id` | uuid | primary key |
| `drop_id` | uuid | indexed |
| `customer_id` | uuid | indexed |
| `status` | text | application enum |
| `quantity` | integer | MVP is 1 |
| `reservation_id` | uuid | unique nullable |
| `payment_id` | uuid | nullable |
| `expires_at` | timestamptz | reservation TTL |
| `created_at` | timestamptz | required |
| `updated_at` | timestamptz | required |

인덱스:

- `idx_orders_customer_created_at`
- `idx_orders_drop_status`
- `idx_orders_expires_at_pending`
- 정책상 필요하면 drop별 customer의 active order를 하나로 제한하는 unique partial index

### `reservations`

| 컬럼 | 타입 | 규칙 |
| --- | --- | --- |
| `id` | uuid | primary key |
| `order_id` | uuid | unique |
| `drop_id` | uuid | indexed |
| `customer_id` | uuid | indexed |
| `quantity` | integer | `> 0` |
| `status` | text | `ACTIVE`, `RELEASED`, `CONFIRMED`, `EXPIRED` |
| `expires_at` | timestamptz | indexed |
| `created_at` | timestamptz | required |
| `updated_at` | timestamptz | required |

### `idempotency_keys`

| 컬럼 | 타입 | 규칙 |
| --- | --- | --- |
| `scope` | text | e.g. `POST /orders` |
| `key` | text | caller가 제공 |
| `request_hash` | text | canonical payload hash |
| `status_code` | integer | 최초 결과 |
| `response_body` | jsonb | 최초 결과 또는 pointer |
| `resource_id` | uuid | order/payment id |
| `created_at` | timestamptz | required |
| `expires_at` | timestamptz | retention |

기본 키:

```sql
PRIMARY KEY (scope, key)
```

### `outbox_events`

| 컬럼 | 타입 | 규칙 |
| --- | --- | --- |
| `id` | uuid | event id |
| `aggregate_type` | text | `order`, `payment`, `catalog` |
| `aggregate_id` | uuid | root id |
| `event_type` | text | topic-compatible type |
| `payload` | jsonb | event payload |
| `headers` | jsonb | trace, causation, idempotency |
| `status` | text | `PENDING`, `PUBLISHED`, `FAILED` |
| `attempt_count` | integer | default 0 |
| `next_attempt_at` | timestamptz | retry schedule |
| `created_at` | timestamptz | required |
| `published_at` | timestamptz | nullable |

인덱스:

- `idx_outbox_pending_next_attempt`
- `idx_outbox_aggregate`
- `idx_outbox_created_at`

## 3. 재고 예약 트랜잭션

`POST /orders` must do the following in one transaction:

1. Insert or read idempotency record.
2. 대상 `inventory_buckets` row를 lock한다.
3. Check drop is open and policy allows purchase.
4. Check `reserved_quantity + confirmed_quantity + requested_quantity <= total_quantity`.
5. Increment `reserved_quantity`.
6. Insert `orders`.
7. Insert `reservations`.
8. Insert `outbox_events` with `order.created`.
9. Save idempotency response.
10. Commit.

SQL 형태:

```sql
UPDATE inventory_buckets
SET reserved_quantity = reserved_quantity + :quantity,
    updated_at = now()
WHERE drop_id = :drop_id
  AND reserved_quantity + confirmed_quantity + :quantity <= total_quantity;
```

이 update는 정확히 1개 row에만 영향을 줘야 한다. 영향받은 row가 0개이면 품절 또는 잘못된 드롭 상태를 반환한다.

## 4. 확정 트랜잭션

When `payment.approved` is consumed:

1. Check `processed_events`.
2. Load order by `payment_id` or event `order_id`.
3. If order is `PENDING_PAYMENT` and not expired, mark reservation `CONFIRMED`.
4. Decrement `reserved_quantity`, increment `confirmed_quantity`.
5. Mark order `CONFIRMED`.
6. Insert `order.confirmed` and `notification.requested` outbox rows.
7. Record processed event.

order가 이미 `CONFIRMED`이면 processed event를 기록하고 success를 반환한다. order가 `EXPIRED` 또는 `CANCELLED`이면 confirm하지 않는다. reconciliation metric을 발행한다.

## 5. 만료 트랜잭션

Reservation expiry worker:

1. Select pending orders where `expires_at < now()`.
2. Lock rows in small batches.
3. Mark reservation `EXPIRED`.
4. Decrement `reserved_quantity`.
5. Mark order `EXPIRED`.
6. Insert `order.reservation.expired` outbox event.

Batch size는 설정 가능해야 한다. Worker lag는 관측 가능해야 한다.

## 6. catalog-service 데이터

핵심 테이블:

- `products`
- `drops`
- `drop_schedules`
- `catalog_outbox_events`
- 선택적 `catalog_read_models`

읽기 모델 필드:

- `drop_id`
- `product_id`
- `title`
- `price`
- `open_at`
- `status`
- `public_badge`
- `cache_tags`
- `updated_at`

읽기 모델에는 최종적 일관성 기반의 `sold_out` 또는 `remaining_hint`가 들어갈 수 있지만, authoritative 값이 아님을 표시해야 한다.

## 7. payment-service 데이터

핵심 테이블:

- `payments`
- `payment_attempts`
- `idempotency_keys`
- `outbox_events`

`payments` should include:

- `id`
- `order_id`
- `customer_id`
- `amount`
- `status`
- `mock_mode`
- `approved_at`
- `failed_at`
- `created_at`
- `updated_at`

## 8. notification-service 데이터

핵심 테이블:

- `notifications`
- `processed_events`
- `dead_letter_events`

`processed_events` 기본 키:

```sql
PRIMARY KEY (consumer_name, event_id)
```

## 9. 보관 정책

| 데이터 | 보관 기간 |
| --- | --- |
| idempotency records | 24 hours minimum, 7 days preferred for demo |
| outbox events | 7 days after published |
| processed events | 30 days |
| dead-letter events | until manual replay or archive |
| traces | environment policy |
| structured logs | environment policy |
