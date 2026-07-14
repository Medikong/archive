# 품절·동시성 테스트 시나리오

작성일: 2026-07-14

실행 결과와 커밋 기준선은 `test-execution-record.md`에 기록한다.

## 1. 현재 필수 시나리오

### SC-01 단일 주문 예약

- Given: 재고가 요청 수량 이상이다.
- When: 주문을 생성한다.
- Then: 201과 `PENDING_PAYMENT` 주문을 반환한다.

### SC-02 재고 부족

- Given: 활성 예약 합계에 새 수량을 더하면 재고를 넘는다.
- When: 주문을 생성한다.
- Then: 주문을 저장하지 않고 409를 반환한다.

### SC-03 PostgreSQL 경쟁

- Given: 재고 1인 상품과 독립 DB 세션 두 개가 있다.
- When: 두 주문 transaction을 동시에 시작한다.
- Then: 한 건만 성공하고 활성 예약은 1이다.

### SC-04 병렬 HTTP와 DB 판정

- Given: 재고 42, 요청 수량 10, 요청 5건이다.
- When: barrier로 요청을 동시에 시작한다.
- Then: 201 네 건, 409 한 건이다.
- And: 활성 주문 네 건과 예약 수량 40을 확인한다.

### SC-05 idempotency

- Given: 같은 사용자와 idempotency key가 있다.
- When: 같은 payload를 반복한다.
- Then: 기존 주문을 반환하고 예약 수량을 다시 늘리지 않는다.

## 2. 후속 시나리오

| ID | 시나리오 | 현재 상태 |
| --- | --- | --- |
| SC-06 | admission 429와 DB 미진입 | 미구현 |
| SC-07 | 장시간 spike p95/p99 | 미구현 |
| SC-08 | 다중 상품 lock 순서 | 미구현 |
| SC-09 | process crash 뒤 outbox 복구 | 미구현 |
| SC-10 | 실제 catalog 재고 원장 연동 | 미구현 |

## 3. 실행 gate 연결

| 검증 | 명령 |
| --- | --- |
| 시나리오 E2E | `task tests:purchase-e2e SCENARIO=06-sold-out-concurrency-flow` |
| 실제 병렬 요청 | `task purchase-e2e-concurrency` |
| 업무 metric | `task purchase-e2e-with-metrics` |
| 전체 내부 회귀 | `task purchase-internal-regression` |

## 4. 공통 판정

- HTTP 응답만 보지 않고 PostgreSQL 최종 상태를 함께 판정한다.
- 성공 예약 합계는 재고를 넘지 않는다.
- 테스트 데이터는 다른 구매 시나리오와 분리한다.
- 실행 뒤 Docker resource를 정리한다.
