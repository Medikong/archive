# 품절·동시성 서비스 구현 계획

작성일: 2026-07-14

경로는 `Medikong/services` 저장소 루트를 기준으로 한다.

## 1. 현재 책임

| 구성요소 | 책임 | 코드 시작점 |
| --- | --- | --- |
| order-service | 주문 생성, lock, 활성 예약 합계, 품절 판정 | `services/order-service/app/postgres.py` |
| order API | 201·409 응답과 idempotency | `services/order-service/app/main.py` |
| catalog-service | 테스트 드롭과 표시 상태 제공 | `services/catalog-service/app/main.py` |
| E2E runner | barrier 병렬 요청과 SQL 최종 판정 | `tests/e2e/scripts/purchase-concurrency-smoke.py` |

## 2. 현재 완료 단위

- 실제 PostgreSQL 독립 세션 경쟁 테스트
- barrier 기반 병렬 HTTP 요청
- 응답 분포와 DB 예약 합계의 이중 판정
- 실패 결제 뒤 예약 합계 회복
- REST idempotency replay와 conflict

## 3. 후속 구현 단위

| 순서 | 작업 | 완료 조건 |
| ---: | --- | --- |
| 1 | 재고 원장 확정 | catalog와 order fixture를 단일 소유권 계약으로 교체한다. |
| 2 | outbox | 주문 commit과 `order.created` 기록이 원자적이다. |
| 3 | admission control | 429 요청이 order transaction에 진입하지 않는다. |
| 4 | 다중 상품 주문 | lock 순서와 교착 회피 정책이 검증된다. |
| 5 | spike SLO | p95/p99, oversell 0, pool·lag 회복을 판정한다. |

## 4. 언어 전환 원칙

order-service를 Go로 전환하더라도 먼저 기존 REST, Kafka, DB 불변조건을 characterization test로 고정한다. Go나 gRPC 자체를 동시성 보장으로 간주하지 않으며, PostgreSQL transaction과 최종 DB 판정을 유지한다.

## 5. 회귀 기준

`pytest services/order-service/tests/integration/postgres_order_concurrency.py`, `task purchase-e2e-concurrency`, `task purchase-internal-regression`을 통과해야 한다.
