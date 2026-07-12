# 품절/동시성 테스트 실행 기록

작성일: 2026-07-07
최종 병렬 주문/DB 재검증: 2026-07-13

이 문서는 품절/동시성 시나리오를 구현하면서 실제로 확인한 테스트와 앞으로 추가해야 할 테스트를 기록한다. 상세 설계는 `00-detailed-design.md`를 기준으로 한다.

## 1. 현재 구현 상태

| 영역 | 현재 상태 |
| --- | --- |
| catalog-service | 품절/동시성 검증 전용 드롭 `drop-sold-out-001`과 상품 `product-sold-out-001`을 제공한다. |
| order-service | 품절/동시성 검증 전용 상품을 주문 가능 카탈로그에 포함한다. |
| 테스트 데이터 | `06-sold-out-concurrency-flow`는 정상 구매용 `drop-001`과 분리된 테스트 드롭을 사용한다. |
| 재고 계산 | `PENDING_PAYMENT`, `CONFIRMED` 주문을 예약/소진 수량으로 보고, `PAYMENT_FAILED` 주문은 예약 수량에서 제외한다. |
| Kafka | 결제 실패/승인 이벤트가 주문 상태와 재고 계산에 영향을 준다. |

## 2. 최근 확인한 테스트

| 계층 | 테스트 | 결과 | 확인 내용 |
| --- | --- | --- | --- |
| unit | `catalog-service` pytest | 통과, 7개 테스트 | 품절/동시성 전용 드롭 상세 조회가 가능하고 request id를 응답에 echo한다. |
| unit | `order-service` pytest | 통과, 21개 테스트 | 품절 응답, 결제 실패 후 재고 release, 결제 승인/실패 이벤트 처리, 업무 metric, request id 응답 echo |
| integration | 실제 PostgreSQL 독립 세션 동시 주문 | 통과, 1개 테스트 | 재고 1에 주문 2건을 동시에 실행해 `OrderCreated` 1건, `ProductSoldOut` 1건, 활성 예약 1건을 확인 |
| e2e | `purchase-e2e-concurrency` | 통과, 병렬 요청 5건 | 수량 10 주문 5건에서 `201` 4건, `409` 1건과 DB 활성 주문 4건/예약 수량 40을 함께 확인 |
| e2e | `06-sold-out-concurrency-flow` Newman | 통과, 8 requests / 15 assertions | 여러 주문으로 재고를 소진한 뒤 초과 주문이 실패하고, 실패 결제 후 재주문 가능 |

## 3. Docker E2E 실행 기록

2026-07-07에 clean Docker Compose 환경에서 품절/동시성 E2E를 다시 실행했다.

| 항목 | 결과 |
| --- | --- |
| 실행 stack | `tests/e2e/docker-compose.yml` |
| 실행 project | `dropmong-purchase-check` |
| 포함 서비스 | postgres, kafka, kafka-init, catalog-service, order-service, payment-service, notification-service |
| Newman 결과 | 8 requests / 15 assertions / failures 0 |
| 평균 응답 시간 | 189ms |
| 확인된 최종 상태 | 재고 소진 후 초과 주문은 409 `product sold out`, 결제 실패 후 재주문 가능 |
| 정리 상태 | 실행 후 compose stack 제거 완료 |

같은 실행에서 `orders_sold_out_total`은 1로 확인했다. 이는 품절/동시성 시나리오의 초과 주문 1건이 정상적으로 거절되었음을 의미한다.

## 4. 시나리오 기준 검증 흐름

```text
POST /orders x 여러 고객
-> 예약 수량 누적
-> POST /orders 초과 수량 요청
-> 409 SOLD_OUT 확인
-> 일부 주문 결제 실패
-> POST /orders 재시도
```

`06-sold-out-concurrency-flow`는 재고 소진과 결제 실패 후 release를 순차적으로 검증한다. 실제 경쟁 상황은 별도 `purchase-e2e-concurrency` gate가 barrier로 5개 요청을 동시에 시작하고, 응답 분포와 PostgreSQL 최종 상태를 함께 판정한다.

## 5. 실제 병렬 주문과 DB 판정 기록

2026-07-13에 clean Compose project `dropmong-oversell-check`에서 실행했다.

| 판정 항목 | 기대값 | 실제 결과 |
| --- | ---: | ---: |
| 동시 주문 수 | 5 | 5 |
| 주문별 수량 | 10 | 10 |
| 성공 응답 | `201` 4건 | `201` 4건 |
| 품절 응답 | `409` 1건 | `409` 1건 |
| DB 활성 주문 | 4건 | 4건 |
| DB 예약 수량 | 40 이하, 재고 42 미초과 | 40 |
| cleanup | container/volume 0 | container/volume 0 |

PostgreSQL repository는 drop/product 조합의 advisory transaction lock을 획득한 뒤 활성 예약 합계를 읽고 주문을 저장한다. 실제 검증에서 oversell이 재현되지 않아 운영 트랜잭션 코드는 변경하지 않았다.

## 6. 실행 명령 기준

서비스 단위 테스트:

```bash
task test-service SERVICE=catalog-service
task test-service SERVICE=order-service
```

품절/동시성 E2E:

```bash
task tests:purchase-e2e SCENARIO=06-sold-out-concurrency-flow
```

실제 병렬 주문과 DB oversell 판정:

```bash
task purchase-e2e-concurrency
```

## 7. 아직 분리해서 추가하면 좋은 테스트

| 계층 | 추가 항목 | 이유 |
| --- | --- | --- |
| integration | 여러 상품을 한 주문에서 예약하는 transaction 테스트 | 현재 검증은 한 상품 단위 advisory lock에 집중한다. |
| e2e | 요청 수와 수량 경계값을 바꾸는 반복 병렬 테스트 | 현재 자동 gate는 재고 42, 수량 10, 요청 5개로 고정되어 있다. |
| load | drop-open spike 테스트 | admission reject, latency p95/p99, oversell count를 관측해야 한다. |
| observability | `oversell_count = 0` 대시보드/알림 | 운영 기준에서 가장 중요한 안전 지표다. |
| observability | sold out/admission metric 확인 | `orders_sold_out_total`은 확인했다. `admission_rejected_total`은 admission control 구현 시 추가한다. |
| observability | Kafka lag 회복 확인 | 테스트 종료 후 주문/결제/알림 consumer lag가 기준 이하로 회복되어야 한다. |

## 8. 완료 판단

품절/동시성 시나리오는 다음 조건을 만족하면 완료로 본다.

- `06-sold-out-concurrency-flow` E2E가 clean compose 환경에서 통과한다.
- 정상 구매/결제 실패 시나리오와 테스트 데이터를 공유하지 않는다.
- 결제 실패로 예약이 풀리면 다음 주문이 가능하다.
- 실제 DB transaction 통합 테스트에서 성공 예약 수가 총 재고를 넘지 않는다.
- 실제 병렬 HTTP 요청과 SQL 판정에서 예약 수량이 총 재고를 넘지 않는다.

위 기능 완료 기준은 충족했다. spike 부하에서의 p95/p99와 운영 `oversell_count` 관측은 성능·운영 hardening 항목으로 남는다.
