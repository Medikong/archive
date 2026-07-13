# 결제 실패 테스트 실행 기록

작성일: 2026-07-07

이 문서는 결제 실패 시나리오를 구현하면서 실제로 확인한 테스트와 앞으로 추가해야 할 테스트를 기록한다. 상세 설계는 `00-detailed-design.md`를 기준으로 한다.

## 1. 현재 구현 상태

| 영역 | 현재 상태 |
| --- | --- |
| order-service | 주문 생성 후 `PENDING_PAYMENT` 상태를 만든다. `payment.failed` 이벤트를 받으면 `PAYMENT_FAILED` 상태로 바꾼다. |
| payment-service | mock 실패 API로 `FAILED` 결제를 만들고 `payment.failed` 이벤트를 발행한다. |
| 재고 처리 | 결제 실패 상태의 주문은 예약 수량 계산에서 제외되어 이후 주문 가능 수량이 회복된다. |
| Kafka | `order.created`로 결제 서비스가 주문을 인식하고, `payment.failed`로 주문 서비스가 실패를 반영한다. |
| 알림 | 실패 알림은 설계 범위에 포함되어 있으나, 현재 핵심 E2E는 주문 상태와 재고 release를 우선 확인한다. |

## 2. 최근 확인한 테스트

| 계층 | 테스트 | 결과 | 확인 내용 |
| --- | --- | --- | --- |
| unit | `order-service` pytest | 통과, 21개 테스트 | `payment.failed` 이벤트 처리, 주문 상태 `PAYMENT_FAILED`, 결제 실패 후 예약 재고 release, 업무 metric, request id 응답 echo |
| unit | `payment-service` pytest | 통과, 22개 테스트 | mock 실패 결제 생성, idempotency, 실패 metric, request id 응답 echo |
| e2e | `05-payment-failure-flow` Newman | 통과, 4 requests / 14 assertions | 주문 생성, 결제 실패, 주문 실패 상태 조회, 실패 후 재고 회복 경로 |

## 3. Docker E2E 실행 기록

2026-07-07에 clean Docker Compose 환경에서 결제 실패 E2E를 다시 실행했다.

| 항목 | 결과 |
| --- | --- |
| 실행 stack | `tests/e2e/docker-compose.yml` |
| 실행 project | `dropmong-purchase-check` |
| 포함 서비스 | postgres, kafka, kafka-init, catalog-service, order-service, payment-service, notification-service |
| Newman 결과 | 4 requests / 14 assertions / failures 0 |
| 평균 응답 시간 | 121ms |
| 확인된 최종 상태 | mock 결제 실패 후 주문 `PAYMENT_FAILED`, 실패 주문은 예약 수량에서 제외 |
| 정리 상태 | 실행 후 compose stack 제거 완료 |

같은 실행에서 `payments_failed_total`은 누적 2로 확인했다. 이 값은 `05`의 결제 실패 1건과 `06`의 재고 release 검증용 실패 1건을 합친 결과다.

## 4. 시나리오 기준 검증 흐름

```text
POST /orders
-> POST /payments/mock-failures
-> GET /orders/{orderId}
-> POST /orders
```

마지막 `POST /orders`는 결제 실패로 예약이 풀린 뒤 다음 주문이 가능한지 확인하는 성격이다.

## 5. 실행 명령 기준

서비스 단위 테스트:

```bash
task test-service SERVICE=order-service
task test-service SERVICE=payment-service
```

결제 실패 E2E:

```bash
task tests:purchase-e2e SCENARIO=05-payment-failure-flow
```

## 6. 아직 분리해서 추가하면 좋은 테스트

| 계층 | 추가 항목 | 이유 |
| --- | --- | --- |
| unit | payment-service의 실패 idempotency conflict 테스트를 시나리오 기준으로 명시 | 같은 결제 키로 다른 payload를 보낼 때 중복 결제 방지가 중요하다. |
| integration | 실제 Kafka를 통한 `payment.failed` consumer 검증 | handler 직접 호출이 아니라 consumer wiring까지 고정해야 한다. |
| integration | 실패 이벤트 중복 수신 idempotency | 중복 이벤트가 주문 상태와 재고 release를 두 번 적용하면 안 된다. |
| e2e | 결제 실패 알림 조회 | 사용자 관점에서는 실패 사유와 다음 행동 안내까지 확인해야 한다. |
| e2e | 결제 지연/예약 만료 | `00-detailed-design.md`에는 포함되어 있으나 현재 자동화는 실패 즉시 처리 중심이다. |
| observability | `payment.failed` trace/log 검색 | 결제 실패 원인과 주문 실패 반영이 같은 correlation으로 추적되어야 한다. |
| observability | 실패 metric 증가 확인 | `payments_failed_total`은 확인했다. `orders_payment_failed_total`, `inventory_released_total`은 추가 구현이 필요하다. |
| observability | PII/raw token 로그 부재 확인 | 실패 로그에 카드 정보, JWT, 개인정보가 남지 않아야 한다. |

## 7. Task 2 실패 기록 (2026-07-13)

Task 2 `결제 실패 중복 이벤트 멱등성`은 첫 ULW 시도에서 실패했으며, 완료로 판단하지 않는다.

| 항목 | 현재 기록 |
| --- | --- |
| payment-service characterization | 실제 PostgreSQL 통합 characterization test는 services 커밋 `3a35578`에 포함되어 있다. |
| 구현 커밋 | payment-service 실제 PostgreSQL characterization test `3a35578`, order-service `processed_payment_events` transactional inbox `387b3da`, HTTP/Kafka/DB E2E gate `b2fd84d` |
| 실행 명령 | `task payment-failure-idempotency`는 Windows bind mount runner를 재설계한 뒤 다시 사용해야 한다. |
| order-service unit regression | 21개 통과 |
| payment-service focused replay unit | 1개 통과 |
| 정적·문법 확인 | Python compile/import, bash syntax, YAML parse, diff check 통과 |
| 실제 PostgreSQL 검증 | C001 `1 passed`: 두 독립 세션에서 `PaymentFailed` 1건, `PaymentAlreadyFailed` 1건, 결제 1행을 확인했다. |
| Docker Kafka E2E | C002 `FAIL`: clean Compose가 healthy 상태까지 기동됐지만 Windows bind mount 하네스에서 세 번 실패했다. 마지막 실패는 `docker compose run`의 미지원 `--mount` 옵션이다. |
| ULW C001/C002/C003 | C001 `PASS`, C002 `FAIL`, C003 미실행. G003은 실패 checkpoint됐다. |
| cleanup | 모든 C002 Compose container/volume과 임시 build context를 제거했다. |
| revert | 검증되지 않은 runner 수정 `63babec`, `6d59f6a`는 `e322a52`, `a95a027`로 되돌렸다. |
| 독립 검토 | review-work 5개 lane은 제한 시간과 후속 요청 후에도 산출물이 없어 모두 `INCONCLUSIVE`이며, 승인으로 간주하지 않는다. |
| coupon service | 이번 Task 2 진행에서 건드리지 않았다. |
| 잔여 위험 | payment-service의 DB commit 이후 Kafka publish 구간은 아직 outbox가 없어 원자성이 보장되지 않으며, 이번 범위 밖이다. |

## 8. 완료 판단

결제 실패 시나리오는 다음 조건을 만족하면 완료로 본다.

- `payment-service`가 실패 결제를 idempotent하게 처리한다.
- `order-service`가 `payment.failed`를 받아 주문을 실패 상태로 전이한다.
- 실패 주문의 예약 수량이 다음 주문 가능 수량에 남지 않는다.
- `05-payment-failure-flow` E2E가 clean compose 환경에서 통과한다.
- 정상 구매 `04`와 품절/동시성 `06`을 이어서 실행해도 상태가 꼬이지 않는다.
