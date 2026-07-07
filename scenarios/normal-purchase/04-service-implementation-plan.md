# 정상 구매 서비스 구현 계획

작성일: 2026-07-03

이 문서는 정상 구매 시나리오를 구현하기 위해 각 서비스에서 필요한 최소 기능, 데이터, 이벤트, 테스트를 정리한다.

## 1. 구현 범위

정상 구매 1차 범위는 다음 네 서비스로 제한한다.

```text
catalog-service
order-service
payment-service
notification-service
```

`auth-service`는 로그인된 사용자 전제로 사용하고, `coupon-service`는 1차 정상 구매에서 제외한다.

## 2. 언어 기준

| 서비스 | 1차 기준 | 성능 검토 |
| --- | --- | --- |
| `catalog-service` | FastAPI | 캐시 전략이 우선이며 언어 전환은 후순위 |
| `order-service` | Go 후보, FastAPI fallback 가능 | 재고 예약과 idempotency 병목이 있으면 Go 유지 또는 전환 |
| `payment-service` | FastAPI | 실제 PG 연동 전까지 mock 결제는 FastAPI로 충분 |
| `notification-service` | FastAPI | Kafka consumer lag가 병목이면 Go worker 검토 |

API와 이벤트 계약은 언어와 독립적으로 유지한다. 언어를 바꿔도 REST path, Kafka topic, payload, metric 이름은 바꾸지 않는다.

## 3. catalog-service

### 책임

- `GET /drops`
- `GET /drops/{dropId}`
- 상품, 가격, 오픈 시각, 구매 제한 정보를 제공한다.
- 공개 조회 응답은 캐시 가능해야 한다.

### 최소 데이터

| 데이터 | 설명 |
| --- | --- |
| `products` | 상품명, 설명, 이미지 |
| `drops` | 상품 연결, 오픈 시각, 상태 |
| `drop_prices` 또는 price snapshot | 판매 가격과 통화 |

### 구현 기준

- `OPEN`, `SCHEDULED`, `SOLD_OUT` 같은 공개 상태를 반환한다.
- 재고 예약 가능 여부를 최종 판단하지 않는다.
- 재고 수량을 보여주더라도 order-service의 예약 결과보다 우선하지 않는다.

### 테스트

- `catalog_drop_list_contract`
- `catalog_drop_detail_contract`
- `catalog_cache_warm_cold_purge`

## 4. order-service

### 책임

- `POST /orders`
- `GET /orders/{orderId}`
- 재고 예약, 주문 생성, idempotency, outbox 저장
- `payment.approved` 소비 후 주문 확정
- `order.confirmed`, `notification.requested` 발행

### 최소 데이터

| 데이터 | 설명 |
| --- | --- |
| `orders` | 주문 상태, 고객, 금액 |
| `order_lines` | 상품/드롭 스냅샷 |
| `inventory_buckets` | drop별 판매 가능 수량 |
| `stock_reservations` | 주문별 예약 수량 |
| `idempotency_keys` | 요청 key와 payload hash, 최초 응답 |
| `outbox_events` | 발행할 Kafka 이벤트 |
| `processed_events` | 중복 consumer 처리 방지 |

### 구현 기준

- 주문 생성과 재고 예약은 하나의 DB transaction으로 처리한다.
- 재고 감소는 `order-service` 내부에서만 일어난다.
- 같은 idempotency key와 같은 payload는 최초 응답을 재사용한다.
- 같은 idempotency key와 다른 payload는 `409 IDEMPOTENCY_KEY_REUSED`를 반환한다.
- `payment.approved`가 중복 도착해도 주문을 한 번만 확정한다.

### 테스트

- `order_create_reserves_inventory`
- `order_idempotency_replay_returns_original_response`
- `order_idempotency_conflict_returns_409`
- `order_create_transaction_prevents_oversell`
- `payment_approved_confirms_order`

## 5. payment-service

### 책임

- `POST /payments`
- mock approve 결제 생성
- payment idempotency
- `payment.approved` outbox 저장과 발행

### 최소 데이터

| 데이터 | 설명 |
| --- | --- |
| `payments` | 결제 상태, 주문 ID, 금액 |
| `payment_idempotency_keys` | 결제 요청 중복 방지 |
| `outbox_events` | `payment.approved` 발행 대기 |

### 구현 기준

- 정상 구매에서는 `simulation = approve`만 사용한다.
- 결제 API 응답은 notification 처리 전에 반환할 수 있다.
- payment는 재고나 주문 확정을 직접 수정하지 않는다.
- `payment.approved`는 결제 사실이고, 합법적 주문 전이는 `order-service`가 판단한다.

### 테스트

- `payment_approve_creates_outbox_event`
- `payment_idempotency_replay_returns_original_response`
- `payment_approved_event_contract`

## 6. notification-service

### 책임

- `notification.requested` 소비
- 주문 확정 알림 저장
- `GET /notifications`
- processed event 기록으로 중복 알림 방지

### 최소 데이터

| 데이터 | 설명 |
| --- | --- |
| `notifications` | 고객별 알림 내용과 읽음 상태 |
| `processed_events` | Kafka 이벤트 중복 처리 방지 |

### 구현 기준

- notification 장애는 checkout 성공 여부에 영향을 주지 않는다.
- 같은 `notification.requested` 이벤트는 알림을 한 번만 만든다.
- 알림이 늦어도 `GET /orders/{orderId}`의 `CONFIRMED` 상태가 구매 완료 기준이다.

### 테스트

- `notification_requested_creates_notification_once`
- `notification_consumer_is_idempotent`
- `notification_after_order_confirmed`

## 7. 구현 순서

1. API와 이벤트 계약을 먼저 고정한다.
2. `catalog-service` 조회 API를 만든다.
3. `order-service` 주문 생성과 재고 예약을 만든다.
4. `payment-service` mock approve와 `payment.approved` 발행을 만든다.
5. `order-service`의 `payment.approved` consumer를 만든다.
6. `notification-service`의 `notification.requested` consumer를 만든다.
7. 정상 구매 E2E를 연결한다.
8. synthetic 또는 load test로 확장한다.

## 8. 완료 기준

- 정상 구매 E2E가 `customer_drop_purchase_happy_path` 이름으로 통과한다.
- 주문 생성은 `PENDING_PAYMENT`를 반환한다.
- 결제 승인 후 주문은 `CONFIRMED`가 된다.
- 주문 확정 이벤트와 알림 요청 이벤트가 outbox를 통해 발행된다.
- 알림 처리가 늦거나 중복되어도 주문 확정은 깨지지 않는다.
