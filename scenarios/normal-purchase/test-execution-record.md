# 정상 구매 테스트 실행 기록

작성일: 2026-07-07

이 문서는 정상 구매 시나리오를 구현하면서 실제로 확인한 테스트와 앞으로 추가해야 할 테스트를 기록한다. 테스트 설계 자체는 `05-test-scenarios.md`를 기준으로 하고, 이 문서는 실행 결과와 현재 구현 상태를 추적한다.

## 1. 현재 구현 상태

| 영역 | 현재 상태 |
| --- | --- |
| catalog-service | `GET /drops`, `GET /drops/{dropId}`로 테스트 드롭과 상품을 조회한다. |
| order-service | `POST /orders`로 주문을 만들고 `PENDING_PAYMENT` 상태와 재고 예약을 만든다. |
| payment-service | mock 승인 API로 `APPROVED` 결제를 만들고 `payment.approved` 이벤트를 발행한다. |
| notification-service | `notification.requested` 이벤트를 받아 고객 알림을 저장하고 조회한다. |
| Kafka | `order.created`, `payment.approved`, `order.confirmed`, `notification.requested` 흐름을 사용한다. |
| 인증 | 로컬 E2E에서는 Gateway JWT 대신 서비스가 신뢰하는 `X-User-*` 테스트 헤더를 사용한다. Gateway JWT 검증은 별도 Gateway E2E 범위다. |

## 2. 최근 확인한 테스트

| 계층 | 테스트 | 결과 | 확인 내용 |
| --- | --- | --- | --- |
| unit | `catalog-service` pytest | 통과, 6개 테스트 | 드롭 목록, 드롭 상세, unknown drop 404, 품절/동시성용 테스트 드롭 조회 |
| unit | `order-service` pytest | 통과, 20개 테스트 | 주문 생성, idempotency replay/conflict, 품절, 결제 승인/실패 이벤트 처리, 결제 실패 후 재고 release, 업무 metric |
| unit | `payment-service` pytest | 통과, 21개 테스트 | 결제 승인/실패 처리, idempotency, 업무 metric |
| e2e | `04-customer-drop-purchase-happy-path` Newman | 통과, 6 requests / 12 assertions | 드롭 조회, 주문 생성, 결제 승인, 주문 확정, 알림 조회 |

## 3. Docker E2E 실행 기록

2026-07-07에 clean Docker Compose 환경에서 구매 시나리오 E2E를 다시 실행했다.

| 항목 | 결과 |
| --- | --- |
| 실행 stack | `tests/e2e/docker-compose.yml` |
| 실행 project | `dropmong-purchase-check` |
| 포함 서비스 | postgres, kafka, kafka-init, catalog-service, order-service, payment-service, notification-service |
| Newman 결과 | 6 requests / 12 assertions / failures 0 |
| 평균 응답 시간 | 57ms |
| 확인된 최종 상태 | 결제 승인 후 주문 `CONFIRMED`, 알림 조회 성공 |
| 정리 상태 | 실행 후 compose stack 제거 완료 |

이어 실행한 `05`, `06` 시나리오까지 포함한 누적 metric은 다음과 같았다.

| Metric | 값 | 해석 |
| --- | ---: | --- |
| `orders_created_total` | 8 | `04` 1건, `05` 1건, `06` 성공 주문 6건 |
| `orders_sold_out_total` | 1 | `06`에서 초과 주문 1건이 품절로 거절됨 |
| `payments_approved_total` | 1 | `04` 정상 구매 결제 승인 1건 |
| `payments_failed_total` | 2 | `05` 결제 실패 1건, `06` 재고 release 검증용 실패 1건 |

## 4. 시나리오 기준 검증 흐름

```text
GET /drops
-> GET /drops/drop-001
-> POST /orders
-> POST /payments/mock-approvals
-> GET /orders/{orderId}
-> GET /notifications
```

## 5. 실행 명령 기준

서비스 단위 테스트:

```bash
task test-service SERVICE=catalog-service
task test-service SERVICE=order-service
task test-service SERVICE=payment-service
task test-service SERVICE=notification-service
```

정상 구매 E2E:

```bash
task tests:purchase-e2e SCENARIO=04-customer-drop-purchase-happy-path
```

## 6. 아직 분리해서 추가하면 좋은 테스트

| 계층 | 추가 항목 | 이유 |
| --- | --- | --- |
| integration | `order.created`가 Kafka를 통해 payment-service에 전달되는지 | 현재 단위 테스트는 handler를 직접 호출하는 성격이 강하다. |
| integration | `payment.approved` 후 order-service가 실제 consumer 경로로 주문을 확정하는지 | 이벤트 payload와 consumer wiring을 함께 고정해야 한다. |
| integration | `notification.requested`가 실제 Kafka consumer로 알림 저장까지 이어지는지 | 비동기 알림 경로가 E2E timeout 원인이 될 수 있다. |
| gateway e2e | JWT 없는 주문 요청 차단 | 로컬 구매 E2E는 Gateway를 우회한다. |
| gateway e2e | 위조 `X-User-*` 헤더 제거 또는 덮어쓰기 | Istio Ingress JWT 구조와 연결되는 보안 테스트다. |
| observability | 정상 구매 journey trace 검색 | 주문 생성부터 결제 승인, 알림까지 `request_id` 또는 `correlation_id`로 추적 가능해야 한다. |
| observability | 주요 metric 증가 확인 | `orders_created_total`, `payments_approved_total`은 확인했다. `notifications_requested_total`은 추가 구현이 필요하다. |
| observability | ERROR log 없음 확인 | synthetic run 동안 관련 서비스에 예상 밖 ERROR 로그가 없어야 한다. |

## 7. 완료 판단

정상 구매 시나리오는 다음 조건을 만족하면 완료로 본다.

- 관련 서비스 단위 테스트가 통과한다.
- `04-customer-drop-purchase-happy-path` E2E가 clean compose 환경에서 통과한다.
- 결제 승인 후 주문 상태가 `CONFIRMED`가 된다.
- 알림 생성 지연이 있어도 구매 완료 자체를 막지 않는다.
- `05-payment-failure-flow`, `06-sold-out-concurrency-flow`를 이어서 실행해도 테스트 데이터 충돌이 없다.
