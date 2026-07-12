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
| 1 | 실제 병렬 주문과 DB oversell 방지 | 진행 중 | 병렬 주문 결과와 PostgreSQL 예약 합계가 재고를 초과하지 않는다. | `sold-out-concurrency/test-execution-record.md` |
| 2 | 결제 실패 중복 이벤트 멱등성 | 대기 | 동일 실패 요청·이벤트가 결제와 주문 상태를 한 번만 변경한다. | `payment-failure/test-execution-record.md` |
| 3 | 구조화 로그 correlation | 대기 | HTTP와 Kafka 경계에서 request/correlation/trace ID를 연결해 검색한다. | `_shared/01-observability-driven-test-contract.md` |
| 4 | Kafka lag 및 notification metric | 대기 | 알림 지표가 증가하고 consumer lag가 기준 이하로 회복된다. | `_shared/01-observability-driven-test-contract.md` |
| 5 | Gateway JWT E2E | 대기 | 유효 JWT만 통과하고 누락·위조 토큰과 위조 사용자 헤더가 차단된다. | `_shared/00-shared-infra-test-contract.md` |
| 6 | 세 시나리오 전체 회귀 테스트 | 대기 | 정상 구매, 결제 실패, 품절/동시성과 모든 신규 gate가 clean 환경에서 통과한다. | `_shared/02-docker-purchase-e2e-runbook.md` |
| 7 | 실행 결과 문서화 | 대기 | 모든 Task의 명령, 결과, 증거, cleanup, 커밋과 잔여 위험이 연결된다. | 이 문서와 각 실행 기록 |

## Task 완료 증거

Task를 완료할 때마다 다음 형식으로 항목을 추가한다.

### Task 1. 실제 병렬 주문과 DB oversell 방지

- 상태: 진행 중
- 기준 구현: `order-service`의 PostgreSQL product advisory transaction lock
- 검증 목표: 재고 42인 상품에 수량 10 주문 5건을 동시에 보내 `201` 4건, `409` 1건을 확인한다.
- DB 판정: 활성 주문 4건, 예약 수량 40, 예약 수량이 42 이하이다.
- 구현 내용: 진행 후 기록
- 실행 명령: 진행 후 기록
- 결과: 진행 후 기록
- 증거: 진행 후 기록
- cleanup: 진행 후 기록
- 커밋: 진행 후 기록
- 남은 위험: 실제 검증 전

## 차단 사항과 후속 작업

| 항목 | 현재 판단 | 다음 행동 |
| --- | --- | --- |
| 쿠폰 서비스 병합 | 현재 시나리오 완성 전까지 보류 | Task 7 이후 별도 병합·연동 계획으로 진행한다. |
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
