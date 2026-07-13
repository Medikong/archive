# 시나리오 개발 진행 현황

작성일: 2026-07-13

이 문서는 정상 구매, 결제 실패, 품절/동시성 시나리오를 실무 기준으로 완성하기 위한 통합 진행판이다. 상세 테스트 증거는 각 시나리오의 `test-execution-record.md`에 기록하고, 이 문서에는 Task 순서, 현재 상태, 완료 증거와 다음 행동을 누적한다.

## 상태 기준

| 상태 | 의미 |
| --- | --- |
| 대기 | 선행 Task가 끝나지 않았거나 아직 시작하지 않았다. |
| 진행 중 | 구현 또는 검증을 수행하고 있다. |
| 차단 | 외부 환경이나 결정이 필요해 진행할 수 없다. |
| 완료 | 구현, 실제 표면 검증, cleanup, 문서 기록이 모두 끝났다. |

## 전체 진행표

| 순서 | Task | 상태 | 완료 기준 | 상세 기록 |
| ---: | --- | --- | --- | --- |
| 1 | 실제 병렬 주문과 DB oversell 방지 | 완료 | 병렬 주문 결과와 PostgreSQL 예약 합계가 재고를 초과하지 않는다. | `sold-out-concurrency/test-execution-record.md` |
| 2 | 결제 실패 중복 이벤트 멱등성 | 진행 중 | 동일 실패 요청·이벤트가 결제와 주문 상태를 한 번만 변경한다. | `payment-failure/test-execution-record.md` |
| 3 | 구조화 로그 correlation | 대기 | HTTP와 Kafka 경계에서 request/correlation/trace ID를 연결해 검색한다. | `_shared/01-observability-driven-test-contract.md` |
| 4 | Kafka lag 및 notification metric | 대기 | 알림 지표가 증가하고 consumer lag가 기준 이하로 회복된다. | `_shared/01-observability-driven-test-contract.md` |
| 5 | Gateway JWT E2E | 대기 | 유효 JWT만 통과하고 누락·위조 토큰과 위조 사용자 헤더가 차단된다. | `_shared/00-shared-infra-test-contract.md` |
| 6 | 세 시나리오 전체 회귀 테스트 | 대기 | 정상 구매, 결제 실패, 품절/동시성과 모든 신규 gate가 clean 환경에서 통과한다. | `_shared/02-docker-purchase-e2e-runbook.md` |
| 7 | 실행 결과 문서화 | 대기 | 모든 Task의 명령, 결과, 증거, cleanup, 커밋과 잔여 위험이 연결된다. | 이 문서와 각 실행 기록 |

## Task 완료 증거

Task를 완료할 때마다 다음 형식으로 항목을 추가한다.

### Task 1. 실제 병렬 주문과 DB oversell 방지

- 상태: 완료
- 기준 구현: `order-service`의 PostgreSQL product advisory transaction lock
- 검증 목표: 재고 42인 상품에 수량 10 주문 5건을 동시에 보내 `201` 4건, `409` 1건을 확인한다.
- DB 판정: 활성 주문 4건, 예약 수량 40, 예약 수량이 42 이하이다.
- 구현 내용: 실제 PostgreSQL 독립 세션 통합 테스트와 barrier 기반 병렬 HTTP smoke를 추가하고, Task가 HTTP 응답 분포와 SQL 예약 합계를 함께 판정하도록 구성했다.
- 실행 명령: `pytest services/order-service/tests/integration/postgres_order_concurrency.py`, `task purchase-e2e-concurrency`, `task purchase-e2e-with-metrics`
- 결과: PostgreSQL 통합 테스트 1개 통과, 병렬 HTTP `201` 4건/`409` 1건, DB `active_orders=4`/`reserved_quantity=40`, `04/05/06/07` Newman failures 0
- 증거: ULW `G002-C001-postgres-integration.txt`, `G002-C002-parallel-http-db.txt`, `G002-C003-purchase-regression.txt`
- cleanup: 전용 Compose project의 container와 volume이 남지 않았음을 각각 확인했다.
- 독립 검토: 코드 리뷰와 수동 QA 모두 `APPROVE`, 차단 이슈 없음
- 커밋: services `ee68f5f`, `3147dae`, `b07e20b`
- 남은 위험: advisory lock은 PostgreSQL 전용이며, 다중 상품 주문과 장시간 spike의 p95/p99 및 `oversell_count` 운영 지표는 별도 부하 테스트가 필요하다.

### Task 2. 결제 실패 중복 이벤트 멱등성

#### Task 2 진행 기록 (2026-07-13)

- 상태: 진행 중
- 현재 확인 사실: payment-service 실제 PostgreSQL 통합 characterization test는 services 커밋 `3a35578`에 포함되어 있다.
- 현재 worktree 사실: order-service의 `processed_payment_events` transactional inbox와 HTTP/Kafka/DB gate 구현은 worktree에 있으나, 이 기록 작성 시점에는 아직 커밋되지 않았다.
- 통과한 로컬 검증: order-service unit regression 21개 통과, payment-service focused replay unit 1개 통과, Python compile/import, bash syntax, YAML parse, diff check 통과.
- 차단된 검증: 실제 PostgreSQL 검증과 Docker Kafka E2E는 Docker API escalation이 현재 환경 사용량 제한으로 거절되어 실행하지 못했다.
- ULW 상태: C001, C002, C003은 pending이며, 실제 PostgreSQL 또는 Docker Kafka E2E PASS 증거를 주장하지 않는다.
- 범위 확인: coupon service는 이번 Task 2 진행에서 건드리지 않았다.
- 남은 위험: payment-service의 DB commit 이후 Kafka publish 구간은 아직 outbox가 없어 원자성이 보장되지 않으며, 이번 범위 밖이다.

## 차단 사항과 후속 작업

| 항목 | 현재 판단 | 다음 행동 |
| --- | --- | --- |
| 쿠폰 서비스 병합 | 현재 시나리오 완성 전까지 보류 | Task 7 이후 별도 병합·연동 계획으로 진행한다. |
| Task 2 실제 PostgreSQL/Kafka E2E | Docker API escalation이 현재 환경 사용량 제한으로 거절되어 실제 PostgreSQL 및 Docker Kafka E2E를 실행하지 못했다. | Docker API 사용이 가능한 환경에서 ULW C001/C002/C003을 다시 실행하고 PASS 증거를 별도 기록한다. |
| Gateway 종류 | 운영 문서는 Kong을 외부 Gateway로 설명하지만 JWT 계약은 Istio를 전제로 한다. | Task 5에서 실제 배포 경로를 먼저 확정한다. |
| Kafka lag metric | 대시보드 후보만 있고 서비스 metric 계약이 없다. | Task 4에서 metric 이름과 측정 책임을 확정한다. |

## 공통 회귀 기준

완료된 Task는 다음 Task 이후에도 유지되어야 한다.

1. `04-customer-drop-purchase-happy-path` failures 0
2. `05-payment-failure-flow` failures 0
3. `06-sold-out-concurrency-flow` failures 0
4. 업무 metric 판정 failures 0
5. HTTP trace와 Kafka span graph 판정 failures 0
6. 각 실행 후 Docker Compose stack과 volume 정리
