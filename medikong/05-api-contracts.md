# DropMong API 계약

작성일: 2026-07-02

이 문서는 구현 전 합의해야 하는 HTTP API 계약을 정의한다. 실제 OpenAPI 파일은 `services/contracts`로 승격한다.

## 1. 공통 규칙

### 헤더

| 헤더 | 필수 여부 | 설명 |
| --- | --- | --- |
| `Authorization: Bearer <jwt>` | 보호 API | customer/operator identity |
| `Idempotency-Key` | 변경 checkout API | 안전한 재시도 |
| `X-Request-Id` | 권장 | caller 또는 gateway request id |
| `traceparent` | propagated | W3C trace context |

### 에러 응답 형식

```json
{
  "error": {
    "code": "SOLD_OUT",
    "message": "Drop is sold out.",
    "request_id": "req_123",
    "details": {}
  }
}
```

### 공통 에러 코드

| HTTP | 코드 | 의미 |
| --- | --- | --- |
| 400 | `INVALID_REQUEST` | request validation 실패 |
| 401 | `UNAUTHENTICATED` | token 없음 또는 token invalid |
| 403 | `FORBIDDEN` | role 불일치 |
| 404 | `NOT_FOUND` | resource 없음 |
| 409 | `IDEMPOTENCY_KEY_REUSED` | 같은 key, 다른 request hash |
| 409 | `SOLD_OUT` | no reservable stock |
| 409 | `ORDER_STATE_CONFLICT` | illegal state transition |
| 422 | `DROP_NOT_OPEN` | drop은 존재하지만 order를 받을 수 없음 |
| 429 | `ADMISSION_REJECTED` | rate limit or queue policy |
| 503 | `TEMPORARILY_UNAVAILABLE` | overloaded dependency or maintenance |

## 2. auth-service

### `POST /auth/login`

요청:

```json
{
  "email": "customer@example.com",
  "password": "password"
}
```

응답:

```json
{
  "access_token": "jwt",
  "token_type": "Bearer",
  "expires_in": 3600,
  "user": {
    "id": "uuid",
    "role": "customer"
  }
}
```

### `GET /auth/me`

JWT가 필요하다. 현재 사용자 id, email, role을 반환한다.

## 3. catalog-service

### `GET /drops`

쿼리:

- `status`
- `cursor`
- `limit`

응답:

```json
{
  "items": [
    {
      "id": "drop_uuid",
      "product_id": "product_uuid",
      "title": "Limited Hoodie",
      "price": 79000,
      "currency": "KRW",
      "status": "SCHEDULED",
      "open_at": "2026-07-10T10:00:00Z",
      "public_badge": "opens-soon"
    }
  ],
  "next_cursor": null
}
```

캐싱:

- 공개 응답은 캐시할 수 있다.
- cache tag `catalog`, `drop:{id}`, `product:{id}`를 포함한다.

### `GET /drops/{dropId}`

공개 드롭 상세를 반환한다. 응답에는 authoritative가 아닌 재고 힌트가 들어갈 수 있지만, 재고 예약 판단에 사용하면 안 된다.

### `POST /admin/drops`

운영자 전용이다. draft drop을 생성한다.

### `POST /admin/drops/{dropId}/schedule`

운영자 전용이다. `open_at`, `close_at`, total quantity seed를 설정한다. 이 작업은 명시적 admin workflow 또는 event를 통해 `order-service` inventory bucket 생성 또는 갱신을 유발해야 한다.

## 4. order-service

### `POST /orders`

필수:

- JWT customer
- `Idempotency-Key`

요청:

```json
{
  "drop_id": "drop_uuid",
  "quantity": 1
}
```

성공 응답:

```json
{
  "order_id": "order_uuid",
  "status": "PENDING_PAYMENT",
  "drop_id": "drop_uuid",
  "quantity": 1,
  "expires_at": "2026-07-02T05:10:00Z",
  "payment_required": true
}
```

규칙:

- 같은 key와 같은 request hash는 같은 status와 body를 반환한다.
- 같은 key와 다른 body는 `409 IDEMPOTENCY_KEY_REUSED`를 반환한다.
- 재고가 없으면 `409 SOLD_OUT`을 반환한다.
- 오픈 상태가 아니면 `422 DROP_NOT_OPEN`을 반환한다.

### `GET /orders/{orderId}`

고객은 자기 주문만 조회할 수 있다. 운영자는 전체 조회가 가능하다.

### `POST /orders/{orderId}/cancel`

MVP 선택 기능이다. payment approval 전까지만 허용한다.

## 5. payment-service

### `POST /payments`

필수:

- JWT customer
- `Idempotency-Key`

요청:

```json
{
  "order_id": "order_uuid",
  "amount": 79000,
  "currency": "KRW",
  "mock_mode": "APPROVE"
}
```

응답:

```json
{
  "payment_id": "payment_uuid",
  "order_id": "order_uuid",
  "status": "APPROVED"
}
```

Mock mode:

- `APPROVE`
- `FAIL`
- `DELAY`

`DELAY`는 reservation TTL과 reconciliation 테스트에 사용한다.

### `GET /payments/{paymentId}`

고객은 자기 결제만 조회할 수 있다. 운영자는 전체 조회가 가능하다.

## 6. notification-service

### `GET /notifications`

현재 사용자의 알림을 반환한다.

### `PATCH /notifications/{notificationId}/read`

알림을 읽음 처리한다.

## 7. 운영 엔드포인트

Every service exposes:

- `GET /healthz`
- `GET /readyz`
- `GET /metrics`

service가 소유한 critical work를 수행할 수 없으면 readiness는 fail해야 한다. readiness로 표현할 수 있는 일시적 dependency error 때문에 liveness가 fail하면 안 된다.

## 8. OpenAPI 전환 목표

구현 시작 후 목표 경로:

```text
services/contracts/services/
  auth-service/openapi.yaml
  catalog-service/openapi.yaml
  order-service/openapi.yaml
  payment-service/openapi.yaml
  notification-service/openapi.yaml
```

기존 `concert-service`, `reservation-service`, `ticket-service` contract를 계속 제자리에서 수정하면 안 된다. DropMong contract가 추가되면 교체하거나 archive해야 한다.
