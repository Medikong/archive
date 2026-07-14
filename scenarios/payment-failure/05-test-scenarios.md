# 결제 실패 테스트 시나리오

작성일: 2026-07-14

이 문서는 수용 동작을 정의한다. 실제 실행 결과는 `test-execution-record.md`에만 기록한다.

## 1. 현재 필수 시나리오

### PF-01 결제 실패 반영

- Given: `PENDING_PAYMENT` 주문이 있다.
- When: mock 실패 결제를 요청한다.
- Then: 결제는 `FAILED`, 주문은 `PAYMENT_FAILED`가 된다.

### PF-02 REST 멱등성

- Given: 동일 사용자와 동일 `Idempotency-Key`가 있다.
- When: 같은 결제 payload를 반복 요청한다.
- Then: 결제 행은 한 건이고 최초 결과를 반환한다.
- And: 다른 payload를 사용하면 409를 반환한다.

### PF-03 Kafka 재전달 멱등성

- Given: 같은 `payment.failed` event ID가 있다.
- When: 이벤트를 세 번 전달한다.
- Then: 처리 event 행과 주문 상태 변경은 각각 한 번만 발생한다.

### PF-04 실패 후 재고 회복

- Given: 상품 재고를 점유한 주문이 결제 실패한다.
- When: 같은 상품에 새 주문을 생성한다.
- Then: 실패 주문은 활성 예약 합계에서 제외되어 재고가 남으면 주문이 성공한다.

## 2. 후속 시나리오

| ID | 시나리오 | 현재 상태 |
| --- | --- | --- |
| PF-05 | 결제 지연 뒤 승인 | 미구현 |
| PF-06 | 결제 지연 뒤 예약 만료 | 미구현 |
| PF-07 | 만료 뒤 늦은 승인 무시 | 미구현 |
| PF-08 | 실패·만료 알림 중복 제거 | 미구현 |
| PF-09 | DB commit 뒤 publish 장애 복구 | 미구현 |
| PF-10 | poison event retry·DLQ | 미구현 |

## 3. 실행 gate 연결

| 검증 | 명령 |
| --- | --- |
| API E2E | `task tests:purchase-e2e SCENARIO=05-payment-failure-flow` |
| HTTP·Kafka·DB 멱등성 | `task payment-failure-idempotency` |
| 로그 correlation | `task tests:purchase-e2e-with-log-correlation` |
| 전체 내부 회귀 | `task purchase-internal-regression` |

## 4. 공통 판정

- 실패를 성공으로 표시하지 않는다.
- 결제·처리 event가 중복 저장되지 않는다.
- 최종 DB 상태와 HTTP 응답을 함께 판정한다.
- 실행 후 Docker container, network, volume과 임시 context가 남지 않는다.
