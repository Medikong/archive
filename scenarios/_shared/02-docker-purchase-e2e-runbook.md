# Docker 구매 E2E 반복 검증 Runbook

작성일: 2026-07-07
최종 Kafka trace E2E 재검증: 2026-07-12
최종 병렬 주문/DB 재검증: 2026-07-13
최종 Loki log correlation 재검증: 2026-07-13

이 문서는 DropMong 구매 시나리오를 로컬 Docker Compose 환경에서 반복 검증하는 기준을 정의한다. 대상은 정상 구매, 결제 실패, 품절/동시성 시나리오다.

## 1. 검증 목적

구매 시나리오 검증은 API 응답만 보는 것이 아니라 다음을 함께 확인한다.

| 항목 | 확인 기준 |
| --- | --- |
| 기능 흐름 | 주문 생성, 결제 승인/실패, 주문 상태 전이, 알림 조회가 기대대로 동작 |
| 메시징 | Kafka 기반 `order.created`, `payment.approved`, `payment.failed`, `order.confirmed`, `notification.requested` 흐름이 깨지지 않음 |
| 테스트 데이터 | 정상 구매, 결제 실패, 품절/동시성 시나리오가 서로의 fixture를 오염시키지 않음 |
| 업무 metric | 성공/실패/품절 지표가 기대 방향으로 증가 |
| trace | 정상 구매 주요 API span과 Kafka producer/consumer span graph가 Tempo에서 고유 request ID로 검색됨 |
| log | HTTP와 Kafka JSON 로그가 Loki에서 correlation/trace ID로 연결되고 민감 필드가 노출되지 않음 |
| 정리 상태 | 실행 후 compose stack과 volume이 제거되어 다음 실행에 영향을 주지 않음 |

## 2. 기본 실행 명령

`services` repo 루트에서 실행한다.

기능 흐름과 업무 metric을 한 번에 검증할 때는 다음 명령을 사용한다.

```bash
task tests:purchase-e2e-with-metrics
```

이 명령은 clean Docker Compose stack에서 `04`, `05`, `06`을 순서대로 실행한 뒤 `07-purchase-flow-metrics` collection으로 `/metrics` 값을 자동 판정한다.

실제 병렬 주문과 PostgreSQL oversell 방지를 확인할 때는 다음 명령을 사용한다.

```bash
task purchase-e2e-concurrency
```

이 명령은 전용 clean stack에서 재고 42인 상품에 수량 10 주문 5건을 barrier로 동시에 보내고, HTTP `201` 4건/`409` 1건과 DB 활성 주문 4건/예약 수량 40을 함께 판정한 뒤 container와 volume을 정리한다.

정상 구매 trace까지 확인할 때는 다음 명령을 사용한다.

```bash
task tests:purchase-e2e-with-traces
```

이 명령은 Tempo와 OpenTelemetry Collector를 포함한 clean Docker Compose stack에서 `04-customer-drop-purchase-happy-path`를 실행한 뒤 `08-purchase-flow-trace-smoke` collection으로 정상 구매 주요 API span을 자동 판정한다.

정상 구매의 Kafka producer/consumer span graph까지 확인할 때는 다음 명령을 사용한다.

```bash
task tests:purchase-e2e-with-kafka-traces
```

이 명령은 clean Docker Compose stack에서 `04-customer-drop-purchase-happy-path`를 실행한 뒤 `09-purchase-kafka-trace-smoke` collection으로 서로 분리된 order root와 payment root를 자동 판정하고 Docker resources를 정리한다.

정상 구매와 결제 실패 로그 연결까지 확인할 때는 다음 명령을 사용한다.

```bash
task tests:purchase-e2e-with-log-correlation
```

이 명령은 clean `git archive` build context에서 stack과 smoke image를 새로 빌드하고, non-root Alloy가 읽기 전용 Docker socket proxy를 통해 현재 Compose project의 JSON stdout만 Loki로 전달하도록 한다. 정상 구매와 결제 실패의 HTTP/Kafka trace 연결, bounded failure code, 민감 필드 부재를 판정한 뒤 container, volume, network와 임시 context를 정리한다.

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

실행 결과는 `services/tests/e2e/newman/reports/`에 JUnit XML로 남는다. 최종 post-security/post-distinct 재실행 XML을 파싱한 결과 `04`는 6 root request executions / 12 testcase entries, `09`는 5 root request executions / 14 testcase entries였고 둘 다 failures 0 / errors 0이었다. Root request execution 수와 assertion별 testcase entry 수는 같은 값으로 해석하지 않는다.

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
| otel-collector | trace 실행 시 OpenTelemetry span 수집 |
| tempo | trace 실행 시 span 저장과 검색 |

## 4. 시나리오별 통과 기준

| 시나리오 | Newman CLI 결과 | 업무 기준 |
| --- | --- | --- |
| `04-customer-drop-purchase-happy-path` | 6 requests / 12 assertions / failures 0 | 결제 승인 후 주문 `CONFIRMED`, 알림 조회 성공 |
| `05-payment-failure-flow` | 4 requests / 14 assertions / failures 0 | 결제 실패 후 주문 `PAYMENT_FAILED`, 예약 수량 release |
| `06-sold-out-concurrency-flow` | 8 requests / 15 assertions / failures 0 | 초과 주문 409 `product sold out`, 실패 결제 후 재주문 가능 |
| `07-purchase-flow-metrics` | 2 requests / 8 assertions / failures 0 | `04/05/06` 이후 order/payment 업무 metric 기대값 확인 |
| `08-purchase-flow-trace-smoke` | 4 requests / 4 assertions / failures 0 | `04` 이후 catalog/order/payment/notification API span 검색 |
| `09-purchase-kafka-trace-smoke` | 최종 재실행 5 requests / 14 assertions / failures 0; polling에 따라 합계 변동 가능 | `04` 이후 서로 다른 order root와 payment root에서 6개 필수 Kafka producer/consumer service/span pair 검색 |
| `10-notification-metrics-happy` | 1 request / 5 assertions / failures 0 | 정상 구매 후 notification counter `1/1/0/0`과 알림 존재 |
| `11-notification-metrics-replay` | 2 requests / 7 assertions / failures 0 | 동일 이벤트 두 건 추가 발행 후 counter `3/2/1/0`, 중복 알림 1건 |
| `purchase-e2e-concurrency` | 병렬 요청 5건, `201` 4건 / `409` 1건 | DB 활성 주문 4건, 예약 수량 40, 재고 42 미초과, cleanup 완료 |

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

## 6. Trace 확인 기준

Trace 확인은 Tempo와 OpenTelemetry Collector가 포함된 Docker stack이 떠 있는 동안만 가능하다. HTTP API span 자동 판정은 `task tests:purchase-e2e-with-traces`, Kafka span graph 자동 판정은 `task tests:purchase-e2e-with-kafka-traces`가 수행한다.

2026-07-07 기준 clean Docker Compose 환경에서 `04-customer-drop-purchase-happy-path` 실행 후 다음 trace 검색을 확인했다.

| 서비스 | 검색 기준 |
| --- | --- |
| `catalog-service` | `service.name=catalog-service request_id=e2e-happy-catalog-list` |
| `order-service` | `service.name=order-service request_id=e2e-happy-order-create` |
| `payment-service` | `service.name=payment-service request_id=e2e-happy-payment-approve` |
| `notification-service` | `service.name=notification-service request_id=e2e-happy-notification-list` |

최종 post-security/post-distinct Kafka trace E2E 재실행에서는 고유 request ID로 현재 실행을 분리했고, `04` Newman CLI 결과는 6 requests / 12 assertions / failures 0, `09` Newman CLI 결과는 5 requests / 14 assertions / failures 0으로 통과했다. `09`의 bounded polling은 Tempo indexing 상태에 따라 검색 request와 해당 assertion을 반복하므로 정확한 requests/assertions 합계는 실행마다 달라질 수 있다. 불변 통과 기준은 failures 0, 서로 다른 order/payment root trace ID, 아래 6개 필수 service/span pair다. 전체 구매 여정은 단일 trace로 판정하지 않는다.

| Trace root | 확인된 span |
| --- | --- |
| order root | `order-service` `kafka.produce order.created`, `payment-service` `kafka.consume order.created` |
| payment root | `payment-service` `kafka.produce payment.approved`, `order-service` `kafka.consume payment.approved`, `order-service` `kafka.produce notification.requested`, `notification-service` `kafka.consume notification.requested` |

검증 후 cleanup exit 0과 잔존 container 0 / volume 0을 확인했다. Log correlation은 `task tests:purchase-e2e-with-log-correlation`로 자동화됐다.

Notification metric과 committed Kafka lag는 `task tests:purchase-e2e-with-notification-metrics`로 확인한다. 앱은 네 business counter를 제공하고, lag는 `notification-service-notification-requested` group의 `notification.requested` offset을 Kafka CLI로 직접 조회한다. 정상 구매의 offset/end/lag는 `1/1/0`, 중복 이벤트 처리 뒤에는 `3/3/0`이다. 실행 후 container, volume, network는 0이고 임시 build context도 남지 않아야 한다.

## 7. 수동 검증이 필요할 때

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

Trace를 수동으로 확인해야 하면 Tempo와 Collector를 함께 띄운 뒤 `04`와 `08`을 이어서 실행한다.

```bash
docker compose -p dropmong-purchase-trace-check -f tests/e2e/docker-compose.yml up -d --build --wait --wait-timeout 180 postgres kafka kafka-init tempo otel-collector catalog-service order-service payment-service notification-service
task tests:purchase-e2e-newman SCENARIO=04-customer-drop-purchase-happy-path E2E_COMPOSE_PROJECT=dropmong-purchase-trace-check E2E_NETWORK=dropmong-purchase-trace-check_default
task tests:purchase-e2e-newman SCENARIO=08-purchase-flow-trace-smoke E2E_COMPOSE_PROJECT=dropmong-purchase-trace-check E2E_NETWORK=dropmong-purchase-trace-check_default
docker compose -p dropmong-purchase-trace-check -f tests/e2e/docker-compose.yml down -v --remove-orphans
```

## 8. 실패했을 때 보는 순서

| 증상 | 먼저 볼 것 |
| --- | --- |
| compose health 실패 | `docker compose ps`, 서비스별 `/readyz` |
| Newman 연결 실패 | compose network 이름, 서비스 URL env-var |
| 주문 생성 실패 | `order-service` log, catalog fixture, `orders_sold_out_total` |
| 결제 승인/실패 반영 실패 | Kafka topic, `payment-service` event 발행, `order-service` consumer |
| 알림 조회 실패 | `notification-service` consumer, `notification.requested` event |
| metric 값 불일치 | clean stack 여부, 시나리오 실행 순서, 이전 volume 잔존 여부 |
| 병렬 주문 빌드 실패 | workspace의 접근 불가 `.pytest_cache`, `.dockerignore`, Docker build context 권한 |
| trace 검색 실패 | `OTEL_TRACES_EXPORTER`, Collector endpoint, Tempo `/api/search`, 요청의 `X-Request-Id` |

## 9. 다음 자동화 대상

현재 `04/05/06` 실행 후 `/metrics`를 자동으로 읽어 기대값을 검증하는 `07-purchase-flow-metrics` collection과 `task tests:purchase-e2e-with-metrics`가 추가되어 있다.

정상 구매 주요 API span은 `08-purchase-flow-trace-smoke` collection과 `task tests:purchase-e2e-with-traces`로 확인한다.

정상 구매 Kafka producer/consumer span graph는 `09-purchase-kafka-trace-smoke` collection과 `task tests:purchase-e2e-with-kafka-traces`로 확인한다. Order root와 payment root는 별도 trace로 판정한다.

Notification 업무 metric과 Kafka lag 자동 판정은 완료했다. 다음 자동화 대상은 Go Task 내장 셸에서도 동작하도록 `09` runner의 `grep` 의존성을 제거하고 `04`부터 `11`까지 한 번에 수행하는 전체 회귀 gate를 만드는 것이다.
