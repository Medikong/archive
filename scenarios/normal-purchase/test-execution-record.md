# 정상 구매 테스트 실행 기록

작성일: 2026-07-07
최종 Kafka trace E2E 재검증: 2026-07-12
최종 내부 통합 재검증: 2026-07-13

이 문서는 정상 구매 시나리오를 구현하면서 실제로 확인한 테스트와 앞으로 추가해야 할 테스트를 기록한다. 테스트 설계 자체는 `05-test-scenarios.md`를 기준으로 하고, 이 문서는 실행 결과와 현재 구현 상태를 추적한다.

## 1. 현재 구현 상태

| 영역 | 현재 상태 |
| --- | --- |
| catalog-service | `GET /drops`, `GET /drops/{dropId}`로 테스트 드롭과 상품을 조회한다. |
| order-service | `POST /orders`로 주문을 만들고 `PENDING_PAYMENT` 상태와 재고 예약을 만든다. |
| payment-service | mock 승인 API로 `APPROVED` 결제를 만들고 `payment.approved` 이벤트를 발행한다. |
| notification-service | `notification.requested` 이벤트를 받아 고객 알림을 저장하고 조회한다. |
| Kafka | 현재 `order.created`, `payment.approved`, `notification.requested` 흐름을 사용한다. `order.confirmed` topic 계약은 예약돼 있지만 아직 발행하지 않는다. |
| 인증 | 로컬 E2E에서는 Gateway JWT 대신 서비스가 신뢰하는 `X-User-*` 테스트 헤더를 사용한다. Gateway JWT 검증은 별도 Gateway E2E 범위다. |

## 2. 최근 확인한 테스트

| 계층 | 테스트 | 결과 | 확인 내용 |
| --- | --- | --- | --- |
| unit | `catalog-service` pytest | 통과, 7개 테스트 | 드롭 목록, 드롭 상세, unknown drop 404, 품절/동시성용 테스트 드롭 조회, request id 응답 echo |
| unit | `order-service` pytest | 통과, 23개 테스트 | 주문 생성, idempotency replay/conflict, 품절, 결제 승인/실패 이벤트 처리, 결제 실패 후 재고 release, 업무 metric, request/correlation id, 이벤트 correlation |
| unit | `payment-service` pytest | 통과, 24개 테스트 | 결제 승인/실패 처리, idempotency, 업무 metric, request/correlation id, 이벤트 correlation |
| unit | `notification-service` pytest | 통과, 18개 테스트 | 알림 조회, Kafka 이벤트 처리, request id 응답 echo, 업무 metric |
| e2e | `04-customer-drop-purchase-happy-path` Newman CLI | 통과, 6 requests / 12 assertions | 드롭 조회, 주문 생성, 결제 승인, 주문 확정, 알림 조회 |
| observability | `08-purchase-flow-trace-smoke` Newman | 통과, 4 requests / 4 assertions | 정상 구매 후 catalog/order/payment/notification 주요 API span을 Tempo에서 검색 |
| observability | `09-purchase-kafka-trace-smoke` Newman CLI | 최종 재실행 통과, 5 requests / 14 assertions / failures 0 | 서로 다른 order root와 payment root에서 6개 필수 Kafka producer/consumer service/span pair를 Tempo에서 검색 |

## 3. Docker E2E 실행 기록

2026-07-07에 clean Docker Compose 환경에서 구매 시나리오 E2E를 다시 실행했다.

| 항목 | 결과 |
| --- | --- |
| 실행 stack | `tests/e2e/docker-compose.yml` |
| 실행 project | `dropmong-purchase-check` |
| 포함 서비스 | postgres, kafka, kafka-init, catalog-service, order-service, payment-service, notification-service |
| Newman CLI 결과 | 6 requests / 12 assertions / failures 0 |
| 평균 응답 시간 | 69ms |
| 확인된 최종 상태 | 결제 승인 후 주문 `CONFIRMED`, 알림 조회 성공 |
| 정리 상태 | 실행 후 compose stack 제거 완료 |

이어 실행한 `05`, `06` 시나리오까지 포함한 누적 metric은 다음과 같았다.

| Metric | 값 | 해석 |
| --- | ---: | --- |
| `orders_created_total` | 8 | `04` 1건, `05` 1건, `06` 성공 주문 6건 |
| `orders_sold_out_total` | 1 | `06`에서 초과 주문 1건이 품절로 거절됨 |
| `payments_approved_total` | 1 | `04` 정상 구매 결제 승인 1건 |
| `payments_failed_total` | 2 | `05` 결제 실패 1건, `06` 재고 release 검증용 실패 1건 |

Trace smoke는 별도 clean Docker Compose 환경에서 `04` 실행 직후 `08-purchase-flow-trace-smoke`를 실행해 확인했다.

| 항목 | 결과 |
| --- | --- |
| 실행 project | `dropmong-purchase-trace-check` |
| 포함 관측성 서비스 | otel-collector, tempo |
| Newman 결과 | 4 requests / 4 assertions / failures 0 |
| 검색 기준 | `service.name` + `request_id` |
| 확인된 서비스 | catalog-service, order-service, payment-service, notification-service |

Kafka trace E2E는 `task tests:purchase-e2e-with-kafka-traces`로 추가 확인했다.

| 항목 | 결과 |
| --- | --- |
| `04-customer-drop-purchase-happy-path` | Newman CLI 결과: 6 requests / 12 assertions / failures 0 |
| `09-purchase-kafka-trace-smoke` | 최종 post-security/post-distinct Newman CLI 결과: 5 requests / 14 assertions / failures 0; bounded polling에 따라 requests/assertions 합계 변동 가능 |
| 실행 격리 | 고유 request ID로 현재 실행의 trace 검색 |
| trace identity | Order root와 payment root trace ID가 서로 다름 |
| order root | `order-service` `kafka.produce order.created`, `payment-service` `kafka.consume order.created` |
| payment root | `payment-service` `kafka.produce payment.approved`, `order-service` `kafka.consume payment.approved`, `order-service` `kafka.produce notification.requested`, `notification-service` `kafka.consume notification.requested` |
| 정리 상태 | Cleanup exit 0, 잔존 container 0 / volume 0 |

`09`는 Tempo indexing을 기다리는 bounded polling에서 검색 request와 그 request의 assertion을 반복할 수 있으므로 정확한 합계가 실행마다 달라질 수 있다. 불변 통과 기준은 failures 0, distinct-root assertion 통과, 두 root에 걸친 6개 필수 pair assertion 통과다. 전체 구매 여정은 단일 trace로 판정하지 않는다.

2026-07-13에는 `task tests:purchase-e2e-with-log-correlation`로 Loki 기반 로그 연결을 추가 검증했다. 정상 구매의 `order.created`, `payment.approved`, `notification.requested` producer/consumer 6개 로그가 동일 `orderId` correlation을 사용했고, 각 로그의 `trace_id`와 `span_id`가 비어 있지 않았다. HTTP 주문·결제 로그는 각각 `request_id`, 같은 값의 `correlation_id`, trace/span ID로 조회됐으며 대응 Kafka 로그와 `trace_id`가 일치했다. 수집기는 non-root Alloy와 읽기 전용 Docker socket proxy를 사용하고 현재 Compose project만 검색한다. 실행 후 container, volume, network는 각각 0이었고 임시 clean build context도 제거됐다. 최종 검증 기준 services 커밋은 `1338150`이다.

같은 날 `task tests:purchase-e2e-with-notification-metrics`로 notification 업무 counter와 Kafka committed lag를 검증했다. 정상 구매 뒤 `consumed/created/replayed/invalid=1/1/0/0`, consumer group offset/end/lag `1/1/0`을 확인했다. 동일 유효 이벤트를 두 번 추가 발행한 뒤에는 `3/2/1/0`, offset/end/lag `3/3/0`, 중복 이벤트 알림 정확히 1건을 확인했다. Newman `10`은 1/5/0, `11`은 2/7/0이다. 당시 전체 회귀에서는 `04`부터 `08`까지 통과했으나 `09` runner가 Go Task 내장 셸의 `grep` 부재로 Newman 전에 중단되어 Task 4를 실패로 기록했다. 구현 기준 services 커밋은 `f7052a4`다.

### 최종 내부 통합 재검증 (G009)

초기 portability 실패 뒤 외부 `grep`/`date` 의존성을 제거하고 UUID로 격리한 깨끗한 Git 추적 파일 clone을 사용하는 `task purchase-internal-regression`을 추가했다. services `ea9710d`에서 8개 gate 통합 실행이 exit 0, 522.6초로 통과했고, 최종 기준선 `34f909b`의 Loki gate도 exit 0, 99.9초로 다시 통과했다.

정상 구매 관련 최종 불변 조건은 HTTP 기능 흐름, purchase metric, HTTP/Kafka trace, Loki correlation, notification metric과 lag가 모두 통과하는 것이다. notification 최종 counter는 `3/2/1/0`, committed lag는 0이며, 정리 뒤 container, network, volume, image와 임시 context는 모두 0이었다. 최종 5개 검토 영역도 모두 `PASS`였다.

이 결과는 Gateway를 제외한 내부 회귀다. 현재는 서비스가 신뢰하는 `X-User-*` header만 사용하며 Istio `RequestAuthentication`/`AuthorizationPolicy`, JWT claim/header mapping, 위조 header 거부, auth-service 통합은 검증하지 않았다. 따라서 운영 또는 외부 ingress 준비 완료를 의미하지 않는다.

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
task tests:purchase-e2e-with-traces
task tests:purchase-e2e-with-kafka-traces
task tests:purchase-e2e-with-log-correlation
task tests:purchase-e2e-with-notification-metrics
```

## 6. 아직 분리해서 추가하면 좋은 테스트

| 계층 | 추가 항목 | 이유 |
| --- | --- | --- |
| integration | `order.created`가 Kafka를 통해 payment-service에 전달되는지 | 현재 단위 테스트는 handler를 직접 호출하는 성격이 강하다. |
| integration | `payment.approved` 후 order-service가 실제 consumer 경로로 주문을 확정하는지 | 이벤트 payload와 consumer wiring을 함께 고정해야 한다. |
| integration | `notification.requested`가 실제 Kafka consumer로 알림 저장까지 이어지는지 | 비동기 알림 경로가 E2E timeout 원인이 될 수 있다. |
| gateway e2e | JWT 없는 주문 요청 차단 | 로컬 구매 E2E는 Gateway를 우회한다. |
| gateway e2e | 위조 `X-User-*` 헤더 제거 또는 덮어쓰기 | Istio Ingress JWT 구조와 연결되는 보안 테스트다. |
| observability | notification metric 증가 확인 | 네 counter와 중복 이벤트 멱등성은 확인했다. 운영 alert threshold는 별도 확정이 필요하다. |
| observability | ERROR log 없음 확인 | synthetic run 동안 관련 서비스에 예상 밖 ERROR 로그가 없어야 한다. |
| observability | Kafka lag 확인 | notification consumer의 committed lag 0 회복은 확인했다. 장시간 spike의 lag SLO는 별도 부하 테스트가 필요하다. |

## 7. 완료 판단

정상 구매 시나리오는 다음 조건을 만족하면 완료로 본다.

- 관련 서비스 단위 테스트가 통과한다.
- `04-customer-drop-purchase-happy-path` E2E가 clean compose 환경에서 통과한다.
- 결제 승인 후 주문 상태가 `CONFIRMED`가 된다.
- 알림 생성 지연이 있어도 구매 완료 자체를 막지 않는다.
- `05-payment-failure-flow`, `06-sold-out-concurrency-flow`를 이어서 실행해도 테스트 데이터 충돌이 없다.
- 정상 구매 주요 API span이 Tempo에서 `service.name` + `request_id`로 검색된다.
- 고유 request ID로 분리된 order root와 payment root에서 정상 구매 Kafka producer/consumer span graph가 검색된다.
