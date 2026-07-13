# 시나리오 개발 진행 현황

작성일: 2026-07-13

이 문서는 정상 구매, 결제 실패, 품절/동시성 시나리오를 실무 기준으로 완성하기 위한 통합 진행판이다. 상세 테스트 증거는 각 시나리오의 `test-execution-record.md`에 기록하고, 이 문서에는 Task 순서, 현재 상태, 완료 증거와 다음 행동을 누적한다.

## 상태 기준

| 상태 | 의미 |
| --- | --- |
| 대기 | 선행 Task가 끝나지 않았거나 아직 시작하지 않았다. |
| 진행 중 | 구현 또는 검증을 수행하고 있다. |
| 차단 | 외부 환경이나 결정이 필요해 진행할 수 없다. |
| 실패 | 검증 시도 한도에 도달해 수정 방향을 다시 설계해야 한다. |
| 완료 | 구현, 실제 표면 검증, cleanup, 문서 기록이 모두 끝났다. |

## 전체 진행표

| 순서 | Task | 상태 | 완료 기준 | 상세 기록 |
| ---: | --- | --- | --- | --- |
| 1 | 실제 병렬 주문과 DB oversell 방지 | 완료 | 병렬 주문 결과와 PostgreSQL 예약 합계가 재고를 초과하지 않는다. | `sold-out-concurrency/test-execution-record.md` |
| 2 | 결제 실패 중복 이벤트 멱등성 | 완료 | 동일 실패 요청·이벤트가 결제와 주문 상태를 한 번만 변경한다. | `payment-failure/test-execution-record.md` |
| 3 | 구조화 로그 correlation | 완료 | HTTP와 Kafka 경계에서 request/correlation/trace ID를 연결해 검색한다. | `_shared/01-observability-driven-test-contract.md` |
| 4 | Kafka lag 및 notification metric | 완료 | 알림 지표가 기대값에 도달하고 실제 consumer lag가 0으로 회복된다. | `_shared/01-observability-driven-test-contract.md` |
| 5 | Gateway JWT E2E | 차단·연기 | 현재 구매 회귀는 신뢰된 내부 `X-User-*` 경계만 검증한다. Gateway JWT와 외부 경계는 별도 배포·인증 통합이 필요하다. | `_shared/00-shared-infra-test-contract.md` |
| 6 | 세 시나리오 전체 회귀 테스트 | 완료 | Gateway를 제외한 8개 gate가 격리된 clean 환경에서 모두 통과한다. | `_shared/02-docker-purchase-e2e-runbook.md` |
| 7 | 실행 결과 문서화 | 완료 | 명령, 결과, 불변 조건, cleanup, 검토 결과와 잔여 위험이 각 실행 기록에 연결된다. | 이 문서와 각 실행 기록 |

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

#### Task 2 실패 기록 (2026-07-13)

- 당시 상태: 실패, 재시도 설계 필요
- 현재 확인 사실: payment-service 실제 PostgreSQL 통합 characterization test는 services 커밋 `3a35578`에 포함되어 있다.
- 구현 커밋: order-service `processed_payment_events` transactional inbox는 services 커밋 `387b3da`에 포함되어 있고, HTTP/Kafka/DB E2E gate는 services 커밋 `b2fd84d`에 포함되어 있다.
- 실행 명령: `task payment-failure-idempotency`는 Windows Git Bash bind mount 문제를 해결한 뒤 다시 사용해야 한다.
- 통과한 로컬 검증: order-service unit regression 21개 통과, payment-service focused replay unit 1개 통과, Python compile/import, bash syntax, YAML parse, diff check 통과.
- 검토 기록: review-work 5개 lane은 bounded wait 이후 deliverable을 반환하지 않아 `INCONCLUSIVE`로 기록한다. 승인이나 PASS로 해석하지 않는다.
- 실제 검증: C001 실제 PostgreSQL 독립 세션 테스트는 `1 passed`로 통과했고 전용 컨테이너를 정리했다.
- 실패 검증: C002는 clean Compose 서비스가 모두 healthy가 된 뒤 Windows bind mount 하네스에서 세 번 실패했다. 마지막 시도는 `docker compose run`이 `--mount`를 지원하지 않아 smoke 실행 전에 종료됐다.
- ULW 상태: C001 `PASS`, C002 `FAIL`, C003 미실행이며 G003은 실패 checkpoint됐다.
- 정리와 revert: 모든 C002 Compose container/volume과 임시 build context를 제거했다. 검증되지 않은 runner 수정 `63babec`, `6d59f6a`는 `e322a52`, `a95a027`로 되돌렸다.
- 로컬 환경 주의: `order-service`와 `payment-service`의 Git-ignored `.pytest_cache`는 샌드박스 전용 ACL 때문에 Docker build context에서 읽을 수 없다. 다음 재시도는 clean `git archive` context를 사용하거나 캐시 권한을 먼저 정리한다.
- 범위 확인: coupon service는 이번 Task 2 진행에서 건드리지 않았다.
- 남은 위험: payment-service의 DB commit 이후 Kafka publish 구간은 아직 outbox가 없어 원자성이 보장되지 않으며, 이번 범위 밖이다.

#### Task 2 재설계 완료 기록 (G008, 2026-07-13)

- 상태: 완료. 위 G003 실패 기록은 당시 실행 사실로 유지하고, 이후 runner와 build context를 재설계해 실제 Docker gate를 통과시켰다.
- 재설계: bind mount와 `git archive`를 사용하지 않는다. `Python` runner가 Git에 추적된 파일만 깨끗한 임시 context로 복사하고, smoke 코드를 image에 포함해 Compose 내부에서 실행한다.
- 검증 결과: 결제 1행, 처리 이벤트 1행, Kafka 재전달 3건의 `eventId` 1개, 주문 최종 상태 `PAYMENT_FAILED`를 확인했다.
- 재고 처리: 실패 주문은 활성 예약 합계에서 제외되는 방식으로 수량을 회복한다.
- 정리 결과: 관련 container, network, volume, image와 임시 context가 모두 0임을 확인했다.
- 남은 범위: `CANCELLED`, `EXPIRED`, 늦은 승인, 실패 알림, transactional outbox는 후속 구현으로 남는다.

### Task 3. 구조화 로그 correlation

- 상태: 완료
- 구현 내용: HTTP 완료 로그에 `correlation_id=request_id`를 추가하고, 구매 Kafka 이벤트의 `correlationId`를 `orderId`로 통일했다. producer/consumer span 안에서 payload를 제외한 구조화 로그를 남긴다.
- 로그 필드: `service.name`, `messaging.operation`, `messaging.destination.name`, partition/offset, `correlation_id`, `trace_id`, `span_id`, `outcome`, 선택적 `failure.code`
- 수집 경로: 컨테이너 stdout JSON -> 읽기 전용 Docker socket proxy -> non-root Grafana Alloy Docker source -> Loki 3.7. Compose project로 실행을 격리하고, 동적 ID는 Loki label로 올리지 않고 JSON field로 검색한다.
- 실행 명령: `task tests:purchase-e2e-with-log-correlation`
- 정상 구매 결과: 동일 `orderId` correlation으로 `order.created`, `payment.approved`, `notification.requested`의 producer/consumer 6개 경계를 확인했다. 주문·결제 HTTP 로그와 대응 Kafka 로그의 `trace_id`도 각각 일치했다.
- 결제 실패 결과: `payment.failed` producer/consumer를 같은 correlation으로 확인했고, 결제 HTTP 로그와 Kafka 로그의 `trace_id`가 일치했다. `failure.code=payment_failed_event`와 raw payload/token/card 필드 부재도 확인했다.
- 보안 판정: token, card, JWT처럼 민감한 모양의 `X-Request-Id`는 UUID로 교체하며, Kafka 로그는 명시적 metadata allowlist만 기록한다.
- 단위 회귀: middleware 11개, kafka-utils 15개, observability 55개, catalog 7개, order 23개, payment 24개, notification 14개 통과
- 전체 회귀: `04` 6/12, `05` 4/14, `06` 8/15, `07` 2/8, `08` 4/4, `09` 5/14, 모든 Newman failures 0
- cleanup: 증거 실행 후 container 0, volume 0, network 0, 임시 clean build context 부재
- 커밋: services `1b1c055`, `0276cbc`, `d539ecf`, `f6ab769`, `2dc792f`, `1338150`, 문서 정합성 `71555fa`
- 증거: ULW `G004-C001-happy-log-correlation.txt`, `G004-C002-failure-log-correlation.txt`, `G004-C003-purchase-regression.txt`
- 독립 검토: 목표 충족, 수동 QA, 코드 품질, 보안, 맥락 검토 5개 lane 모두 `PASS`, 차단 이슈 없음
- 범위 확인: coupon service는 수정하거나 병합하지 않았다.
- 남은 위험: Docker socket proxy와 Alloy Docker discovery는 로컬 E2E 전용이다. 운영 환경의 수집 권한, Loki 보존 기간, tenant/auth, alert 정책은 infra 배포 설계에서 별도로 확정해야 한다.

### Task 4. Kafka lag 및 notification metric

- 상태: 완료. 아래 초기 runner 실패를 수정한 G009 통합 실행에서 최종 통과했다.
- 구현 내용: `notification-service`에 이벤트 소비, 알림 생성, 중복 재수신, 잘못된 이벤트 counter를 추가했다. 동적 ID는 metric label로 사용하지 않는다.
- lag 책임: 애플리케이션이 자체 추정 gauge를 내보내지 않는다. 실제 Kafka consumer group `notification-service-notification-requested`의 `notification.requested` committed offset을 조회해 lag를 판정한다.
- 실행 명령: `task purchase-e2e-with-notification-metrics`
- 정상 구매 결과: Newman `04` 6 requests / 12 assertions / failures 0, metric `consumed=1`, `created=1`, `replayed=0`, `invalid=0`, 알림 존재, offset/end/lag `1/1/0`, Newman `10` 1/5/0
- 중복 이벤트 결과: 같은 유효 이벤트를 두 번 추가 발행한 뒤 metric `3/2/1/0`, 중복 이벤트 알림 정확히 1건, offset/end/lag `3/3/0`, Newman `11` 2/7/0
- 초기 실패 기록: notification 단위 테스트와 `04`~`08`은 통과했지만, 원본 context의 `.pytest_cache` ACL, 기존 stack의 `18084` 충돌, `09` runner의 `grep` 의존성으로 전체 gate는 완료되지 못했다.
- 최종 결과: notification counter `consumed/created/replayed/invalid=3/2/1/0`, 중복 이벤트 알림 1건, consumer lag 0을 확인했다.
- cleanup: 관련 Compose project의 container, volume, network는 모두 0이고 임시 clean clone context도 제거했다.
- 구현 커밋: services `4164553`, `7923597`, `f7052a4`
- 증거: ULW `G005-C001-notification-metrics-lag.txt`, `G005-C002-notification-replay-lag.txt`, `G005-C003-purchase-regression.txt`
- 범위 확인: coupon service는 수정하거나 병합하지 않았다.
- portability 수정: 외부 `grep`과 `date` 의존성을 제거하고 Python과 POSIX 내장 판정으로 교체했다.

### Task 5. Gateway JWT E2E

- 상태: 차단·연기
- 현재 경계: 구매 서비스는 내부에서 전달된 `X-User-Id`, `X-User-Email`, `X-User-Role`을 신뢰하며, 이번 통합 실행도 그 내부 경계만 검증한다.
- 제외 범위: Istio `RequestAuthentication`/`AuthorizationPolicy`, JWT claim과 사용자 header mapping, 외부 위조 header 거부, auth-service 통합은 구현·검증하지 않았다.
- 판정: 이번 결과를 운영 또는 외부 ingress 준비 완료로 해석하지 않는다.

### Task 6. 세 시나리오 전체 회귀 테스트

- 상태: 완료, Gateway 제외
- services 기준선: `34f909b`
- 통합 실행: services `ea9710d`에서 `task purchase-internal-regression`, exit 0, 522.6초
- 8개 검증: unit, purchase metric, 실제 concurrency, 결제 실패 중복 멱등성, HTTP trace, Kafka trace, Loki correlation, notification metric/lag
- 핵심 불변 조건: 동시 주문 `201` 4건/`409` 1건, 활성 주문 4건, 예약 수량 40이 재고 42 이하
- 결제 실패 불변 조건: 결제 1행, 처리 이벤트 1행, Kafka 3건의 `eventId` 1개, 주문 `PAYMENT_FAILED`
- notification 불변 조건: counter `3/2/1/0`, consumer lag 0
- 최종 log gate: services `34f909b`에서 exit 0, 99.9초
- 정리 결과: container, network, volume, image, 임시 context 모두 0
- 독립 검토: 최종 5개 검토 영역 모두 `PASS`

### Task 7. 실행 결과 문서화

- 상태: 완료
- 기록 대상: 이 진행판, 공통 Docker runbook, 정상 구매·결제 실패·품절/동시성 실행 기록
- 기록 원칙: G003 실패를 보존하고 G008 `Python` runner와 깨끗한 Git 추적 파일 복제 방식의 성공을 후속 기록으로 추가했다.
- 범위 경고: Gateway JWT, 운영 ingress, 미구현 결제 상태와 transactional outbox를 완료로 표현하지 않는다.

## 차단 사항과 후속 작업

| 항목 | 현재 판단 | 다음 행동 |
| --- | --- | --- |
| 쿠폰 서비스 병합 | 이번 구매 내부 회귀와 분리되어 있다. | 별도 병합·연동 계획으로 진행한다. |
| Gateway JWT | 내부 `X-User-*` 신뢰 경계만 검증했고 외부 위조 요청은 검증하지 않았다. | Istio 정책, claim/header mapping, 위조 header 거부와 auth-service 통합을 별도 Gateway E2E로 수행한다. |
| 결제 후속 상태 | 현재 수용 동작은 `PAYMENT_FAILED`와 예약 집계 제외 방식이다. | `CANCELLED`, `EXPIRED`, 늦은 승인, 실패 알림을 별도 상태 전이 설계로 추가한다. |
| 이벤트 원자성 | payment-service의 DB commit과 Kafka publish 사이 transactional outbox가 없다. | outbox 도입과 장애 복구 검증을 별도 Task로 수행한다. |

## 공통 회귀 기준

완료된 Task는 다음 Task 이후에도 유지되어야 한다.

1. `04-customer-drop-purchase-happy-path` failures 0
2. `05-payment-failure-flow` failures 0
3. `06-sold-out-concurrency-flow` failures 0
4. 업무 metric 판정 failures 0
5. HTTP trace와 Kafka span graph 판정 failures 0
6. 각 실행 후 Docker Compose stack과 volume 정리
