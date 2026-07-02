# DropMong 제품 범위

작성일: 2026-07-02

이 문서는 DropMong 전환의 제품 범위와 구현 경계를 고정한다. 목적은 기존 Ticketmong 형태의 좌석 예매 시스템을 제한 수량 드롭 커머스로 바꿀 때, 무엇을 만들고 무엇을 만들지 않을지 먼저 합의하는 것이다.

## 1. 제품 정의

DropMong은 정해진 오픈 시각에 한정 수량 상품을 판매하는 드롭 커머스다.

고객 관점에서는 "오픈 예정 상품을 보고, 오픈 시간이 되면 빠르게 주문하고 결제하는 서비스"다. 운영자 관점에서는 "상품, 드롭 일정, 판매 수량을 관리하고 피크 트래픽과 장애 상태를 확인하는 서비스"다. 플랫폼 관점에서는 "짧은 시간에 몰리는 주문을 제한된 재고 안에서 정확하게 확정하고, 실패한 결제와 지연된 이벤트를 복구 가능한 형태로 운영하는 시스템"이다.

DropMong은 다음 문제를 풀기 위해 존재한다.

| 문제 | DropMong의 답 |
| --- | --- |
| 오픈 순간 사용자가 몰린다. | admission control로 주문 경로에 들어오는 요청을 제한한다. |
| 같은 사용자가 재시도하면 중복 주문이 생길 수 있다. | `Idempotency-Key`와 request hash로 같은 요청을 같은 결과로 묶는다. |
| 한정 수량보다 많이 팔리면 안 된다. | `order-service`가 재고 예약과 주문 상태를 같은 transaction에서 처리한다. |
| 결제, 알림, 이벤트가 지연될 수 있다. | transactional outbox, Kafka, idempotent consumer로 복구 가능한 비동기 흐름을 만든다. |
| 배포 중 장애가 날 수 있다. | Argo Rollouts canary와 SLO 기반 rollback으로 위험한 변경을 점진 배포한다. |

### 서비스 경험

```text
오픈 전: 고객은 드롭 목록과 상세를 본다.
오픈 순간: gateway와 admission layer가 요청을 받아들이거나 빠르게 거절한다.
주문 생성: order-service가 재고를 예약하고 pending order를 만든다.
결제: payment-service가 승인, 실패, 지연 결과를 이벤트로 발행한다.
확정: order-service가 payment 결과를 소비해 주문을 확정하거나 예약을 해제한다.
알림: notification-service가 확정, 실패, 품절, 운영 이벤트를 비동기로 처리한다.
```

핵심 문장:

```text
드롭 오픈 순간에 트래픽이 몰려도 판매 가능 수량보다 많은 주문이 확정되면 안 된다.
```

## 2. 핵심 목표

| 목표 | 설명 | 검증 기준 |
| --- | --- | --- |
| oversell 0 | 확정 주문 수가 판매 가능 수량을 넘지 않는다. | 부하 테스트 후 `oversell_count = 0` |
| 피크 트래픽 방어 | 드롭 오픈 순간 요청을 무제한으로 order path에 넣지 않는다. | admitted, queued, rejected RPS 분리 |
| 안전한 재시도 | 클라이언트, gateway, payment 재시도에도 중복 주문이 생기지 않는다. | 같은 `Idempotency-Key` replay 결과 동일 |
| 장애 격리 | 알림, catalog projection, Kafka 일부 지연이 주문 확정을 막지 않는다. | consumer 중단 중 핵심 흐름 성공 |
| 점진 배포 | order/payment 변경은 canary와 자동 rollback으로 배포한다. | SLO 위반 시 rollback 동작 |

## 3. 사용자와 역할

| 역할 | 주요 행동 | 필요한 기능 |
| --- | --- | --- |
| 비회원 | 드롭 목록과 상세를 조회한다. | 공개 catalog 조회 |
| 고객 | 로그인, 주문 생성, 결제, 주문 상태 확인을 한다. | JWT, order, payment |
| 운영자 | 드롭 생성, 수량 설정, 오픈 시각 설정, 장애 상태 확인을 한다. | admin catalog, observability |
| 플랫폼 운영자 | 배포, rollback, scaling, incident 대응을 한다. | GitOps, dashboard, runbook |

## 4. MVP 범위

MVP에 포함한다:

- 회원 로그인과 JWT 기반 인증
- 상품과 드롭 목록, 상세 조회
- 드롭 오픈 시각과 판매 수량 설정
- 주문 생성과 재고 예약
- 예약 TTL 만료와 재고 release
- mock payment 승인, 실패, 지연
- 결제 결과에 따른 주문 확정 또는 취소
- 주문 및 결제 이벤트의 transactional outbox
- Kafka 기반 알림 요청
- Prometheus, Grafana, Loki, Tempo 기반 관측
- k6 기반 피크 트래픽과 oversell 검증
- Istio Ingress Gateway 기반 외부 진입
- Argo Rollouts 기반 canary와 rollback

MVP에서 제외한다:

- 실제 PG 연동
- 추천 알고리즘
- 복잡한 쿠폰, 포인트, 프로모션
- 다중 창고 재고
- 정산 시스템
- 배송사 연동
- 별도 inventory-service
- 별도 waiting-room-service

제외한 기능은 나중에 붙일 수 있지만, 1차 구현에서는 oversell 0과 운영 검증을 흐리면 안 된다.

## 5. 서비스 범위

서비스는 5개로 시작한다.

| 서비스 | 제품 책임 |
| --- | --- |
| `auth-service` | 로그인, JWT, role claim |
| `catalog-service` | 상품, 드롭, 공개 조회, catalog projection |
| `order-service` | 주문 생성, 재고 예약, 예약 TTL, 주문 상태 |
| `payment-service` | mock payment 승인, 실패, 지연, 결제 이벤트 |
| `notification-service` | 사용자 알림, 운영 알림, 이벤트 소비 |

서비스를 늘리지 않는 이유:

- `order-service`가 재고와 주문을 함께 소유해야 oversell 0을 단순하게 검증할 수 있다.
- `ticket-service`는 드롭 커머스의 핵심 도메인이 아니며 주문 확정 결과로 흡수한다.
- `inventory-service`는 다중 창고, 복수 판매 채널, 외부 재고 연동이 생길 때 분리한다.
- `waiting-room-service`는 트래픽이 admission layer 수준을 넘어설 때 분리한다.

## 6. 주요 유스케이스

### UC-01 드롭 조회

고객은 오픈 전후에 드롭 목록과 상세를 조회한다.

성공 기준:

- 공개 GET은 캐시 가능한 응답이다.
- catalog read 지연이 order path에 영향을 주지 않는다.
- 재고 진실은 catalog cache에 두지 않는다.

### UC-02 주문 생성

고객은 오픈된 드롭에 주문을 생성한다.

성공 기준:

- `POST /orders`는 `Idempotency-Key`를 요구한다.
- `order-service`는 같은 트랜잭션에서 order, reservation, idempotency record, outbox를 쓴다.
- 수량 부족 시 명확한 `409 SOLD_OUT` 또는 `409 INSUFFICIENT_STOCK`을 반환한다.

### UC-03 결제 처리

고객은 pending order에 대해 결제를 요청한다.

성공 기준:

- `POST /payments`는 `Idempotency-Key`를 요구한다.
- payment 승인 이벤트는 order 확정으로 이어진다.
- payment 실패 또는 timeout은 reservation release로 이어진다.

### UC-04 장애 격리

notification consumer가 내려가도 주문과 결제는 성공해야 한다.

성공 기준:

- notification lag 또는 DLQ가 증가한다.
- order/payment error rate는 영향을 받지 않는다.
- 복구 후 알림 재처리가 가능하다.

## 7. 제품 불변 조건

| 불변 조건 | 소유 문서 | 구현 위치 |
| --- | --- | --- |
| 확정 주문 수는 판매 수량을 넘지 않는다. | `04-data-design.md` | `order-service` DB transaction |
| 같은 idempotency key와 같은 payload는 같은 결과를 반환한다. | `05-api-contracts.md` | `order-service`, `payment-service` |
| 같은 idempotency key와 다른 payload는 거부한다. | `05-api-contracts.md` | API middleware 또는 service layer |
| 이벤트 발행 실패는 DB 변경을 잃게 만들지 않는다. | `06-event-contracts.md` | transactional outbox |
| 알림 장애는 핵심 checkout을 막지 않는다. | `07-critical-flows.md` | Kafka consumer isolation |
| 배포 실패는 자동 rollback 가능해야 한다. | `08-infra-deployment.md` | Argo Rollouts analysis |

## 8. 저장소 영향

| 저장소 | 범위 |
| --- | --- |
| `workspaces` | DropMong 설계, ADR, 구현 계획, 합의 문서 |
| `services` | 서비스명, API 계약, DB schema, event producer/consumer 변경 |
| `e-gitops` | Helm values, Istio, Rollouts, KEDA, synthetic test 변경 |
| `infra` | project name, ECR repository, ingress/NLB wiring 변경 |
| `archive` | 이전 Ticketmong 문서와 결정 로그 보관 후보 |

## 9. 완료 정의

제품 범위 설계 완료는 다음 조건을 만족해야 한다.

- 5개 서비스 책임이 문서로 고정되어 있다.
- order-owned inventory 결정이 ADR로 남아 있다.
- API와 이벤트 계약이 구현 전에 합의되어 있다.
- oversell 0, retry storm, payment failure, notification outage 테스트가 정의되어 있다.
- HTML 보드는 문서 읽는 순서와 핵심 흐름만 보여준다.
