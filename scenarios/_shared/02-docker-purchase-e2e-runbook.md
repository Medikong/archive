# Docker 구매 E2E 반복 검증 Runbook

작성일: 2026-07-07

이 문서는 DropMong 구매 시나리오를 로컬 Docker Compose 환경에서 반복 검증하는 기준을 정의한다. 대상은 정상 구매, 결제 실패, 품절/동시성 시나리오다.

## 1. 검증 목적

구매 시나리오 검증은 API 응답만 보는 것이 아니라 다음을 함께 확인한다.

| 항목 | 확인 기준 |
| --- | --- |
| 기능 흐름 | 주문 생성, 결제 승인/실패, 주문 상태 전이, 알림 조회가 기대대로 동작 |
| 메시징 | Kafka 기반 `order.created`, `payment.approved`, `payment.failed`, `order.confirmed`, `notification.requested` 흐름이 깨지지 않음 |
| 테스트 데이터 | 정상 구매, 결제 실패, 품절/동시성 시나리오가 서로의 fixture를 오염시키지 않음 |
| 업무 metric | 성공/실패/품절 지표가 기대 방향으로 증가 |
| 정리 상태 | 실행 후 compose stack과 volume이 제거되어 다음 실행에 영향을 주지 않음 |

## 2. 기본 실행 명령

`services` repo 루트에서 실행한다.

기능 흐름과 업무 metric을 한 번에 검증할 때는 다음 명령을 사용한다.

```bash
task tests:purchase-e2e-with-metrics
```

이 명령은 clean Docker Compose stack에서 `04`, `05`, `06`을 순서대로 실행한 뒤 `07-purchase-flow-metrics` collection으로 `/metrics` 값을 자동 판정한다.

개별 시나리오만 확인할 때는 다음 명령을 사용한다.

```bash
task tests:purchase-e2e SCENARIO=04-customer-drop-purchase-happy-path
task tests:purchase-e2e SCENARIO=05-payment-failure-flow
task tests:purchase-e2e SCENARIO=06-sold-out-concurrency-flow
```

세 시나리오를 한 번에 확인할 때는 `SCENARIO`를 생략한다.

```bash
task tests:purchase-e2e
```

실행 결과는 `services/tests/e2e/newman/reports/`에 JUnit XML로 남는다.

## 3. 실행 대상 stack

구매 E2E는 다음 Docker Compose stack을 사용한다.

```text
tests/e2e/docker-compose.yml
```

기본 포함 서비스는 다음과 같다.

| 서비스 | 목적 |
| --- | --- |
| postgres | 서비스별 테스트 DB |
| kafka | 이벤트 전파 |
| kafka-init | 구매 시나리오 topic 초기화 |
| catalog-service | drop, product 조회 |
| order-service | 주문 생성, 재고 예약, 주문 상태 전이 |
| payment-service | mock 결제 승인/실패 |
| notification-service | 구매 결과 알림 저장/조회 |

## 4. 시나리오별 통과 기준

| 시나리오 | Newman 기준 | 업무 기준 |
| --- | --- | --- |
| `04-customer-drop-purchase-happy-path` | 6 requests / 12 assertions / failures 0 | 결제 승인 후 주문 `CONFIRMED`, 알림 조회 성공 |
| `05-payment-failure-flow` | 4 requests / 14 assertions / failures 0 | 결제 실패 후 주문 `PAYMENT_FAILED`, 예약 수량 release |
| `06-sold-out-concurrency-flow` | 8 requests / 15 assertions / failures 0 | 초과 주문 409 `product sold out`, 실패 결제 후 재주문 가능 |
| `07-purchase-flow-metrics` | 2 requests / 8 assertions / failures 0 | `04/05/06` 이후 order/payment 업무 metric 기대값 확인 |

## 5. Metric 확인 기준

Metric 확인은 Docker stack이 떠 있는 동안만 가능하다. 자동 판정은 `task tests:purchase-e2e-with-metrics`가 수행한다. 직접 확인할 때는 6장의 수동 검증 흐름처럼 stack을 유지한 상태에서 Newman을 실행한 뒤 다음 endpoint를 본다.

```text
order-service:   http://127.0.0.1:18085/metrics
payment-service: http://127.0.0.1:18086/metrics
```

세 시나리오를 순서대로 실행했을 때 2026-07-07 기준 확인한 값은 다음과 같다.

| Metric | 기대값 | 의미 |
| --- | ---: | --- |
| `orders_created_total` | 8 | `04` 1건, `05` 1건, `06` 성공 주문 6건 |
| `orders_sold_out_total` | 1 | `06`의 초과 주문 1건 |
| `order_idempotency_replay_total` | 0 | 이번 E2E에서는 idempotency replay 없음 |
| `order_idempotency_conflict_total` | 0 | 이번 E2E에서는 idempotency conflict 없음 |
| `payments_approved_total` | 1 | `04` 정상 구매 결제 승인 1건 |
| `payments_failed_total` | 2 | `05` 실패 1건, `06` release 검증용 실패 1건 |

이 값은 clean compose 환경에서 `04`, `05`, `06`을 순서대로 실행했을 때의 누적값이다. 단일 시나리오만 실행하면 기대값이 달라진다.

## 6. 수동 검증이 필요할 때

Task 실행이 막히거나 metric을 직접 확인해야 하면 다음 흐름으로 본다.

```bash
docker compose -p dropmong-purchase-check -f tests/e2e/docker-compose.yml up -d --build --wait --wait-timeout 180 postgres kafka kafka-init catalog-service order-service payment-service notification-service
task tests:purchase-e2e-newman SCENARIO=04-customer-drop-purchase-happy-path E2E_COMPOSE_PROJECT=dropmong-purchase-check E2E_NETWORK=dropmong-purchase-check_default
task tests:purchase-e2e-newman SCENARIO=05-payment-failure-flow E2E_COMPOSE_PROJECT=dropmong-purchase-check E2E_NETWORK=dropmong-purchase-check_default
task tests:purchase-e2e-newman SCENARIO=06-sold-out-concurrency-flow E2E_COMPOSE_PROJECT=dropmong-purchase-check E2E_NETWORK=dropmong-purchase-check_default
task tests:purchase-e2e-newman SCENARIO=07-purchase-flow-metrics E2E_COMPOSE_PROJECT=dropmong-purchase-check E2E_NETWORK=dropmong-purchase-check_default
docker compose -p dropmong-purchase-check -f tests/e2e/docker-compose.yml down -v --remove-orphans
```

수동 실행에서는 마지막 `down -v --remove-orphans`까지 완료해야 다음 실행이 같은 조건에서 시작된다.

## 7. 실패했을 때 보는 순서

| 증상 | 먼저 볼 것 |
| --- | --- |
| compose health 실패 | `docker compose ps`, 서비스별 `/readyz` |
| Newman 연결 실패 | compose network 이름, 서비스 URL env-var |
| 주문 생성 실패 | `order-service` log, catalog fixture, `orders_sold_out_total` |
| 결제 승인/실패 반영 실패 | Kafka topic, `payment-service` event 발행, `order-service` consumer |
| 알림 조회 실패 | `notification-service` consumer, `notification.requested` event |
| metric 값 불일치 | clean stack 여부, 시나리오 실행 순서, 이전 volume 잔존 여부 |

## 8. 다음 자동화 대상

현재 `04/05/06` 실행 후 `/metrics`를 자동으로 읽어 기대값을 검증하는 `07-purchase-flow-metrics` collection과 `task tests:purchase-e2e-with-metrics`가 추가되어 있다.

다음 단계는 `tests/e2e/observability`의 Tempo trace smoke를 구매 서비스에 연결해 `request_id` 또는 `correlation_id` 기준으로 정상 구매 journey trace를 확인하는 것이다.
