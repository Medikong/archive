# 관측성 기반 실전형 테스트 계약

작성일: 2026-07-07

이 문서는 DropMong 시나리오 테스트를 실제 운영에 가깝게 만들기 위한 관측성 기반 검증 기준을 정의한다. 기존 E2E가 API 응답과 상태 전이를 확인한다면, 관측성 기반 테스트는 같은 흐름이 trace, metric, log, Kafka lag, alert 기준에서도 추적 가능하고 안전한지 확인한다.

## 1. 현재 repo 기준

`services` repo에는 이미 로컬 관측성 E2E 구조가 있다.

```text
tests/e2e/observability/
  docker-compose.yml
  otel-collector.yml
  tempo.yml
  grafana/
  scripts/trace-smoke.py
```

현재 자동화 범위는 다음과 같다.

| 항목 | 현재 상태 |
| --- | --- |
| trace backend | Docker Compose로 OpenTelemetry Collector, Tempo, Grafana 실행 |
| 자동 판정 | `trace-smoke.py`가 Tempo API를 polling해서 trace 존재 여부 확인 |
| 대상 서비스 | 현재 기본값은 `coupon-service` |
| 제외 endpoint | `/healthz`, `/readyz`, `/metrics`는 trace에 남지 않아야 함 |
| 구매 시나리오 서비스 | `catalog-service`, `order-service`, `payment-service`, `notification-service`는 아직 구매 journey trace E2E와 직접 연결되어 있지 않음 |
| 구매 서비스 metric | `order-service`, `payment-service`는 구매 업무 metric을 일부 제공한다. `notification-service`, Kafka lag, trace 연계는 추가 작업이 필요하다. |

따라서 구매 시나리오의 실전형 테스트는 기존 E2E에 관측성 검증을 붙이는 방향으로 확장한다.

## 2. 실전형 테스트의 목표

실전형 테스트는 다음 질문에 답해야 한다.

```text
기능이 성공했는가?
-> 장애가 났을 때 어디서 막혔는지 찾을 수 있는가?
-> 트래픽이 몰려도 잘못된 성공이 발생하지 않는가?
-> 실패/지연/중복 이벤트가 운영 지표에 드러나는가?
-> 배포 중 문제가 생기면 자동으로 멈출 수 있는가?
```

## 3. 공통 실행 단위

모든 실전형 테스트는 고유 실행 ID를 가진다.

```text
synthetic_run_id: syn-20260707-001
X-Request-Id: req-syn-20260707-001
Idempotency-Key: idem-syn-20260707-001
```

이 값은 다음 위치에서 찾아야 한다.

| 위치 | 확인 기준 |
| --- | --- |
| HTTP response | `X-Request-Id`가 응답에 echo된다. |
| Tempo trace | `request_id`, `service.name`, `http.route`로 검색 가능하다. |
| Loki log | `request_id` 또는 `correlation_id`로 검색 가능하다. |
| Kafka header | `traceparent`, `tracestate`, `correlation_id`가 유지된다. |
| 업무 이벤트 | `eventId`, `orderId`, `paymentId` 같은 업무 식별자로 상태를 대조할 수 있다. |

## 4. 관측성 기반 테스트 계층

| 계층 | 이름 | 목적 |
| --- | --- | --- |
| observability smoke | trace 수집 경로 확인 | 서비스 요청 span이 Collector와 Tempo까지 도달하는지 확인 |
| journey trace test | 사용자 흐름 trace 확인 | 한 시나리오의 주요 API와 Kafka 경계가 trace/log로 이어지는지 확인 |
| metric assertion test | 운영 지표 확인 | 성공/실패/품절/lag/oversell 지표가 기대값을 만족하는지 확인 |
| log correlation test | 장애 분석 가능성 확인 | 같은 `request_id` 또는 `correlation_id`로 관련 로그를 찾을 수 있는지 확인 |
| failure injection test | 장애 상황 확인 | 결제 실패, Kafka 지연, notification 장애, DB lock 상황을 넣고 복구 경로 확인 |
| load + SLO test | 운영 부하 확인 | drop-open spike에서 latency, error rate, oversell, lag 기준 확인 |
| release gate test | 배포 안전 확인 | canary 중 Prometheus/Tempo/Loki 기준이 나쁘면 rollout 중단 |

## 5. 정상 구매 관측성 테스트

정상 구매는 다음을 확인한다.

| 검증 | 기준 |
| --- | --- |
| trace | `GET /drops`, `POST /orders`, `POST /payments/mock-approvals`, `GET /orders/{orderId}`, `GET /notifications` 요청 span이 있다. |
| event trace | `order.created`, `payment.approved`, `order.confirmed`, `notification.requested` 경계가 trace 또는 log correlation으로 이어진다. |
| metric | `orders_created_total`, `payments_approved_total`, `notifications_requested_total`이 증가한다. |
| latency | 정상 구매 journey 전체 p95가 목표 시간 이하이다. |
| log | 같은 `request_id` 또는 `correlation_id`로 주문 생성부터 결제 승인까지 찾을 수 있다. |
| error | 해당 synthetic run 동안 관련 서비스에 ERROR 로그가 없어야 한다. |

최종 판정:

```text
API E2E 성공
+ 주문 상태 CONFIRMED
+ 알림 조회 성공
+ trace 검색 가능
+ ERROR log 없음
+ 주요 metric 증가
```

## 6. 결제 실패 관측성 테스트

결제 실패는 실패가 “정상적으로 실패했는지”를 관측성으로 확인해야 한다.

| 검증 | 기준 |
| --- | --- |
| trace | 결제 실패 요청과 주문 실패 반영 요청/consumer 처리가 검색 가능하다. |
| event | `payment.failed` event id가 log 또는 trace attribute로 남는다. |
| metric | `payments_failed_total`, `orders_payment_failed_total`, `inventory_released_total`이 증가한다. |
| stock | 실패 주문 수량이 예약 수량에 남지 않는다. |
| log | 실패 사유는 안전한 code로 남고 raw 카드 정보, JWT, 개인정보는 남지 않는다. |
| alert | 실패율이 기준 이상일 때 alert 조건에 걸릴 수 있어야 한다. |

최종 판정:

```text
05 E2E 성공
+ 주문 상태 PAYMENT_FAILED
+ 예약 수량 release
+ payment.failed 추적 가능
+ 실패 사유 metric/log 확인
+ PII 로그 없음
```

## 7. 품절/동시성 관측성 테스트

품절/동시성은 기능 성공보다 “초과 판매가 없었는지”가 핵심이다.

| 검증 | 기준 |
| --- | --- |
| metric | `oversell_count`가 항상 0이다. |
| metric | `orders_sold_out_total`과 `orders_created_total`이 구분되어 증가한다. |
| latency | drop-open spike 중 `POST /orders` p95/p99가 목표 범위 안에 있다. |
| Kafka lag | 주문/결제/알림 consumer lag가 테스트 종료 후 기준 이하로 회복된다. |
| trace | 품절 응답과 성공 예약 요청을 각각 검색할 수 있다. |
| log | sold out, idempotency replay, conflict, admission reject가 code로 구분된다. |

최종 판정:

```text
06 E2E 성공
+ 성공 예약 수 <= 총 재고
+ oversell_count = 0
+ sold out metric 증가
+ Kafka lag 회복
+ p95/p99 기준 만족
```

## 8. 실패 주입 테스트 후보

| 주입 상황 | 기대 결과 |
| --- | --- |
| payment-service가 `payment.failed` 발행 | order-service가 주문 실패와 재고 release를 처리한다. |
| notification-service 일시 중단 | 구매 확정은 성공하고 알림만 재시도 또는 지연된다. |
| Kafka publish 실패 | outbox pending 또는 retry metric이 증가하고 요청은 정해진 정책대로 실패/보류된다. |
| Kafka consumer 중복 수신 | 주문 확정, 실패 처리, 알림 생성이 중복 적용되지 않는다. |
| order-service DB lock 지연 | timeout, retry, sold out, admission reject가 구분되어 기록된다. |
| 잘못된 JWT 또는 위조 헤더 | Gateway에서 차단되고 서비스 내부 로그에 raw token이 남지 않는다. |

## 9. 필요한 metric 초안

구매 시나리오에 필요한 최소 metric은 다음과 같다.

| Metric | Type | 기준 |
| --- | --- | --- |
| `orders_created_total` | counter | 주문 생성 성공 수 |
| `orders_confirmed_total` | counter | 주문 확정 수 |
| `orders_payment_failed_total` | counter | 결제 실패로 실패 처리된 주문 수 |
| `orders_sold_out_total` | counter | 품절 응답 수 |
| `order_create_duration_seconds` | histogram | 주문 생성 latency |
| `payments_approved_total` | counter | 결제 승인 수 |
| `payments_failed_total` | counter | 결제 실패 수 |
| `notification_requested_total` | counter | 알림 요청 수 |
| `kafka_consumer_lag` | gauge | consumer lag |
| `outbox_pending_count` | gauge | 발행 대기 이벤트 수 |
| `oversell_count` | gauge | 항상 0이어야 하는 안전 지표 |

동적 ID는 Prometheus label로 올리지 않는다. `order_id`, `payment_id`, `customer_id`, `synthetic_run_id`는 log와 trace 검색에만 사용한다.

## 10. 구현 순서

1. 구매 서비스의 `/metrics`를 `service_ready`에서 업무 metric까지 확장한다.
2. `catalog-service`, `order-service`, `payment-service`, `notification-service`에 FastAPI trace instrumentation을 붙인다.
3. E2E 요청에 고유 `X-Request-Id`와 `synthetic_run_id`를 넣는다.
4. Kafka producer/consumer header에 `traceparent`, `tracestate`, `correlation_id`를 전파한다.
5. `tests/e2e/observability`의 trace smoke를 구매 서비스에도 적용할 수 있게 파라미터화한다.
6. `04/05/06` Newman 실행 뒤 Tempo, Prometheus, Loki 또는 log output을 조회하는 검증 스크립트를 추가한다.
7. 품절/동시성은 순차 E2E와 별도로 병렬 주문 스크립트 또는 k6 테스트를 추가한다.
8. canary/rollout에서는 `oversell_count > 0`, error rate 급등, p95 초과, Kafka lag 미회복을 중단 조건으로 사용한다.

## 11. 현재 바로 가능한 테스트

지금 바로 실행 가능한 관측성 테스트는 기존 Go 서비스 trace smoke다.

```bash
task tests:test-observability-e2e
```

구매 시나리오의 기능 E2E와 order/payment 업무 metric 자동 확인은 `02-docker-purchase-e2e-runbook.md` 기준으로 바로 실행할 수 있다. trace 기반 관측성 테스트까지 연결하려면 다음 작업이 먼저 필요하다.

- 구매 Python 서비스에 trace exporter 설정
- 구매 E2E compose에 Collector/Tempo 연결
- Newman 실행 후 Tempo search 검증 스크립트 추가
- notification 업무 metric, Kafka lag metric, oversell metric 추가

즉, 현재 구매 시나리오는 기능 E2E와 order/payment 업무 metric 자동 검증까지 갖췄고, trace/log/Kafka lag까지 포함한 실전형 관측성 자동화는 다음 개발 task로 분리하는 것이 맞다.
