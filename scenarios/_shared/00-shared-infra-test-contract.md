# Shared Infra & Test Contract

작성일: 2026-07-03

이 문서는 DropMong 시나리오 오너들이 공통으로 따라야 하는 인프라, API, 이벤트, 테스트 계약을 정의한다. 각 시나리오 문서는 이 문서를 기준으로 작성하며, 공통 계약을 바꿔야 하는 경우 이 문서를 먼저 수정한다.

## 1. 목적

두 명이 서로 다른 시나리오를 구현하더라도 같은 실행 환경에서 통합될 수 있도록 다음 항목을 고정한다.

- 서비스 이름
- API path
- Kafka topic
- DB 소유권
- 인증과 추적 헤더
- 상태값
- 테스트 계층과 테스트 이름
- 관측성과 릴리즈 기준

## 2. 공통 서비스 이름

| 서비스 | 역할 | 기본 언어 |
| --- | --- | --- |
| `auth-service` | 로그인, JWT 발급, 사용자 식별 | FastAPI |
| `catalog-service` | 상품, drop, 공개 조회 | FastAPI |
| `order-service` | 주문, 재고 예약, idempotency, order outbox | Go 후보 |
| `payment-service` | mock 결제, payment outbox | FastAPI |
| `notification-service` | 알림 저장, notification consumer | FastAPI |
| `coupon-service` | 쿠폰 발급, 사용 예약/확정 | Go 후보 |
| `message-recovery-service` | DLQ, replay, 운영 복구 | FastAPI |

과도기 구현에서 기존 `concert-service`, `reservation-service` 이름을 사용하더라도 DropMong 설계 문서와 신규 계약은 `catalog-service`, `order-service` 이름을 기준으로 한다.

## 3. 공통 API 경로

| API | 소유 서비스 | 사용 시나리오 |
| --- | --- | --- |
| `POST /auth/login` | `auth-service` | 인증 및 회원 |
| `GET /auth/me` | `auth-service` | 인증 및 회원 |
| `GET /drops` | `catalog-service` | 정상 구매, 품절/동시성 |
| `GET /drops/{dropId}` | `catalog-service` | 정상 구매, 품절/동시성 |
| `POST /orders` | `order-service` | 정상 구매, 품절/동시성, 결제 실패 |
| `GET /orders/{orderId}` | `order-service` | 정상 구매, 결제 실패 |
| `POST /payments` | `payment-service` | 정상 구매, 결제 실패 |
| `GET /payments/{paymentId}` | `payment-service` | 결제 실패, 운영 확인 |
| `GET /notifications` | `notification-service` | 정상 구매, 결제 실패, 알림 확인 |
| `POST /coupons/issues` | `coupon-service` | 쿠폰 |
| `GET /coupons/issues` | `coupon-service` | 쿠폰 |

외부 API는 REST + JSON + OpenAPI로 정의한다.

## 4. 공통 Kafka Topic

DropMong 신규 이벤트 이름은 dot notation을 사용한다.

| Topic | Producer | Consumer | 목적 |
| --- | --- | --- | --- |
| `order.created` | `order-service` | `payment-service` | 결제 대상 주문을 `known_orders`에 등록 |
| `payment.approved` | `payment-service` | `order-service` | 결제 승인 |
| `payment.failed` | `payment-service` | `order-service` | 결제 실패 |
| `payment.delayed` | 미구현 | 미구현 | 후속 결제 지연용 예약 계약 |
| `order.confirmed` | 미구현 | 미구현 | 후속 분석·확장용 예약 계약 |
| `order.cancelled` | 미구현 | 미구현 | 후속 주문 취소 알림용 예약 계약 |
| `order.reservation.expired` | 미구현 | 미구현 | 후속 예약 만료 알림용 예약 계약 |
| `notification.requested` | `order-service`, `coupon-service` | `notification-service` | 알림 생성 요청 |
| `coupon.issued` | `coupon-service` | `notification-service` | 쿠폰 발급 |
| `coupon.usage.confirm.requested` | `order-service` | `coupon-service` | 쿠폰 사용 확정 요청 |
| `coupon.usage.release.requested` | `order-service` | `coupon-service` | 쿠폰 사용 해제 요청 |

이벤트 payload에는 업무 필드만 넣고, trace 전파 값은 Kafka header를 사용한다.

## 5. DB 소유권

| DB | 소유 서비스 | 다른 서비스 직접 접근 |
| --- | --- | --- |
| `auth-db` | `auth-service` | 금지 |
| `catalog-db` | `catalog-service` | 금지 |
| `order-db` | `order-service` | 금지 |
| `payment-db` | `payment-service` | 금지 |
| `notification-db` | `notification-service` | 금지 |
| `coupon-db` | `coupon-service` | 금지 |
| `message-recovery-db` | `message-recovery-service` | 금지 |

규칙:

- 다른 서비스의 DB를 직접 조회하거나 수정하지 않는다.
- 상태 변경은 API 또는 Kafka event로 전달한다.
- DB migration은 해당 서비스 구현 repo가 소유하고, 배포 순서는 GitOps runbook이 소유한다.

## 6. 공통 헤더

| 헤더 | 필수 여부 | 설명 |
| --- | --- | --- |
| `Authorization` | 외부 보호 API 필수 | `Bearer <JWT>` |
| `X-Request-Id` | 필수 | 요청 추적 |
| `traceparent` | 권장 필수 | W3C trace context |
| `Idempotency-Key` | 생성/변경 API 필수 | 중복 요청 방지 |

`POST /orders`, `POST /payments`, `POST /coupons/issues`는 `Idempotency-Key`를 요구한다.

### 6.1 Istio Gateway JWT 정책

DropMong 목표 ingress에서는 Kong plugin이 아니라 Istio Ingress Gateway가 JWT 검증의 1차 책임을 가진다. 클라이언트 API 계약은 계속 `Authorization: Bearer <JWT>`를 사용한다.

| 항목 | 기준 |
| --- | --- |
| JWT 검증 위치 | Istio Ingress Gateway |
| JWT 인증 리소스 | `RequestAuthentication` |
| 보호 API 강제 | `AuthorizationPolicy` + `requestPrincipals` |
| 공개 API | `POST /auth/login`, `GET /drops`, `GET /drops/{dropId}` |
| 보호 API | `GET /auth/me`, `POST /orders`, `GET /orders/{orderId}`, `POST /payments`, `GET /notifications`, coupon/admin API |

`RequestAuthentication`은 잘못된 JWT를 거부하지만, JWT가 없는 요청을 보호 API에서 자동으로 막는 기준으로 보면 부족하다. 보호 API는 `AuthorizationPolicy`에서 인증된 principal을 요구하도록 별도 선언한다.

현재 구매 서비스가 사용하는 내부 사용자 context는 다음 두 헤더다. 최종 claim/header mapping은 Istio Gateway와 auth-service owner가 함께 확정한다.

| 내부 헤더 | JWT claim | 설명 |
| --- | --- | --- |
| `X-User-Id` | `sub` | 서비스 내부 사용자 식별자 |
| `X-User-Role` | `role` | 현재 구매 런타임은 `CUSTOMER`, `OWNER`, `ADMIN`을 사용 |

인증 계약에는 아직 두 가지 드리프트가 있다. `services/contracts/jwt-conventions.md`와 order OpenAPI는 선택적 `email` claim 및 `X-User-Email`을 정의하지만 현재 구매 런타임은 이 헤더를 소비하지 않는다. 또한 JWT/OpenAPI 역할값은 `CUSTOMER`, `OPERATOR`, `ADMIN`인데 현재 구매 런타임은 `CUSTOMER`, `OWNER`, `ADMIN`이다. auth-service owner와 Gateway 계약을 확정할 때 이메일 전파 여부와 `OPERATOR`/`OWNER` 중 사용할 역할값을 함께 결정하며, 결정 전에는 어느 쪽도 완료된 통합 계약으로 보지 않는다.

서비스는 외부 클라이언트가 직접 보낸 `X-User-*` 값을 신뢰하지 않는다. Istio Gateway는 검증된 JWT claim으로 해당 헤더를 덮어쓰거나, 보호 API로 전달되기 전에 위조 헤더를 제거해야 한다. 내부 서비스 간 호출은 mTLS와 전파된 사용자 context를 함께 사용하며, `Authorization` 원문은 로그에 남기지 않는다.

## 7. 공통 상태값

| 도메인 | 상태 |
| --- | --- |
| drop | `SCHEDULED`, `OPEN`, `SOLD_OUT`, `CLOSED` |
| order | 목표 계약은 `PENDING_PAYMENT`, `CONFIRMED`, `CANCELED`, `EXPIRED`. 현재 구매 자동 검증은 `PAYMENT_FAILED`를 추가로 사용 |
| payment | `REQUESTED`, `APPROVED`, `FAILED`, `DELAYED` |
| notification | `PENDING`, `SENT`, `FAILED`, `READ` |
| coupon issue | `ISSUED`, `SOLD_OUT`, `REVOKED` |
| coupon usage | `RESERVED`, `CONFIRMED`, `RELEASED`, `EXPIRED` |

상태값은 시나리오별로 임의 확장하지 않는다. 새 상태가 필요하면 shared 문서와 API/event 계약을 함께 수정한다.

## 8. 통신 방식

| 통신 | 기준 |
| --- | --- |
| 외부 API | REST + OpenAPI |
| 서비스 간 단순 동기 조회 | REST |
| 상태 변경 전파 | Kafka |
| 내부 고성능 동기 호출 | 초기 제외, 필요 시 gRPC ADR 작성 |

초기 구현에서는 gRPC를 사용하지 않는다.

## 9. 공통 인프라 기준

| 영역 | 기준 |
| --- | --- |
| 배포 단위 | 서비스별 Helm release |
| GitOps | `e-gitops`의 `charts/medikong-service`와 `values/services/*` |
| 이미지 | 서비스 repo가 build/push, GitOps가 tag 반영 |
| 외부 진입 | 과도기 Kong, 목표 Istio Gateway + `RequestAuthentication`/`AuthorizationPolicy` |
| 메시징 | Kafka |
| 관측성 | Prometheus, Loki, Tempo |
| 데이터 | 서비스별 DB, local/dev에서는 platform data chart 사용 가능 |

시나리오 문서는 인프라 리소스를 직접 새로 정의하지 않고 이 기준 위에서 필요한 값을 제안한다.

## 10. 테스트 계층

| 계층 | 위치 | 목적 |
| --- | --- | --- |
| unit | `services/<service>/tests` | domain state transition, idempotency, event handler |
| contract | `services/contracts` | OpenAPI request/response, error envelope |
| integration | `services/<service>/tests/integration` | DB transaction, outbox relay, consumer idempotency |
| e2e | `services/tests/e2e/scenarios` | Docker Compose 기반 사용자 흐름 |
| synthetic | `e-gitops/platform/synthetic` | 배포된 환경 smoke와 journey |
| load | `e-gitops/platform/loadtest` | drop-open traffic, overload |
| release | `e-gitops` | canary, rollback |

각 시나리오 오너는 자신의 구현에 대해 unit, integration, e2e를 함께 추가한다.

## 11. 공통 E2E 이름

| 시나리오 | E2E 이름 |
| --- | --- |
| 정상 구매 | `customer_drop_purchase_happy_path` |
| 품절/동시성 | `customer_sold_out_concurrency_path` |
| 결제 실패 | `customer_payment_failed_path` |
| 결제 지연/예약 만료 | `customer_payment_delayed_expiry_path` |
| 쿠폰 발급 | `customer_coupon_issue_path` |
| 쿠폰 적용 구매 | `customer_coupon_purchase_path` |
| 운영자 drop 준비 | `operator_drop_setup_smoke` |

## 12. 공통 로컬 검증 순서

```bash
task test-service SERVICE=<service-name>
task test-unit
task test-e2e SCENARIO=<scenario-name>
```

DropMong 전환 전 기존 티켓팅 이름을 사용하는 경우에도 신규 테스트 이름은 DropMong 시나리오 이름으로 작성한다.

구매 시나리오 전용 Docker E2E는 다음 명령을 기준으로 한다.

```bash
task tests:purchase-e2e-with-metrics
task tests:purchase-e2e-with-traces
task tests:purchase-e2e SCENARIO=04-customer-drop-purchase-happy-path
task tests:purchase-e2e SCENARIO=05-payment-failure-flow
task tests:purchase-e2e SCENARIO=06-sold-out-concurrency-flow
```

반복 실행, metric 확인, 정상 구매 trace smoke 기준은 `02-docker-purchase-e2e-runbook.md`를 따른다.

## 13. 관측성 기준

모든 서비스는 다음 endpoint를 제공한다.

```text
GET /healthz
GET /readyz
GET /metrics
```

모든 structured log는 다음 값을 포함한다.

```text
service
event
request_id
trace_id
correlation_id
```

동적 ID는 metric label로 사용하지 않는다. `order_id`, `payment_id`는 접근이 통제되고 보존 기간이 제한된 log와 trace에서만 사용할 수 있다. 원본 `customer_id`는 linkable identifier이므로 log와 trace에도 남기지 않으며, 업무상 필요하면 별도 salt와 접근 정책을 가진 pseudonymous subject key를 사용한다.

관측성 기반 실전형 테스트 기준은 `01-observability-driven-test-contract.md`를 따른다. 기능 E2E가 통과해도 trace, metric, log, Kafka lag로 원인을 추적할 수 없으면 운영 준비 완료로 보지 않는다.

## 14. 릴리즈 게이트

서비스 변경 merge 전:

- unit test 통과
- contract test 통과
- DB migration 하위 호환
- idempotency test 통과
- event handler duplicate test 통과

외부 route 활성화 전:

- health/readiness 통과
- internal smoke 통과
- JWT 없음, 잘못된 JWT, 권한 부족, 정상 JWT gateway test 통과
- 위조 `X-User-*` 헤더가 제거되거나 검증된 claim으로 덮어써지는지 확인
- metric scrape 확인
- log와 trace 확인

100 percent rollout 전:

- oversell 없음
- accepted order latency가 SLO 이하
- outbox pending과 Kafka lag 안정
- canary analysis 통과

## 15. 변경 규칙

- shared 문서의 API, topic, DB, 상태값을 바꾸려면 영향을 받는 시나리오 문서를 함께 수정한다.
- 한 시나리오의 편의를 위해 공통 계약을 임의로 바꾸지 않는다.
- 기존 계약을 깨는 변경은 migration 또는 compatibility plan을 포함한다.
- 언어 변경은 공통 API, event, DB 소유권 변경 사유가 아니다.
