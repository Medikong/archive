# DropMong 보안 설계

작성일: 2026-07-02

이 문서는 인증, 인가, 서비스 간 통신, secret, network policy, audit 기준을 정의한다.

## 1. 보안 목표

- 고객은 본인 주문과 알림만 볼 수 있다.
- 운영자 API는 role claim과 network policy로 제한한다.
- 서비스 간 통신은 mTLS와 최소 권한을 사용한다.
- secret은 Git에 plaintext로 저장하지 않는다.
- 장애 대응에 필요한 audit log는 남기되 민감 정보는 남기지 않는다.

## 2. 인증

외부 API:

- JWT Bearer token 사용
- `auth-service`가 token 발급
- gateway 또는 service에서 JWT validation
- service는 user id와 role claim을 신뢰 가능한 context로 받는다.

필수 claim:

| Claim | 설명 |
| --- | --- |
| `sub` | user id |
| `role` | `customer`, `operator`, `admin` |
| `iat` | issued at |
| `exp` | expiration |
| `iss` | issuer |

## 3. 인가

| 작업 | 필요 role | 소유권 규칙 |
| --- | --- | --- |
| 공개 drop 조회 | anonymous 허용 | 없음 |
| 주문 생성 | customer | 자기 user id |
| 주문 조회 | customer 또는 operator | customer는 자기 주문만 조회 |
| 결제 생성 | customer | 자기 order만 가능 |
| drop 관리 | operator 또는 admin | operator scope |
| DLQ 조회 | operator 또는 admin | internal only |
| rollout admin | 플랫폼 운영자 | cluster RBAC |

JWT가 identity를 제공하는 경우 service는 request body의 customer id를 신뢰하면 안 된다.

## 4. 서비스 간 보안

목표:

- 단계적 rollout 후 Istio mTLS STRICT 적용
- service-level access를 위한 AuthorizationPolicy
- internal admin endpoint는 외부에 노출하지 않음

Policy 예시:

| 출발지 | 목적지 | 허용 범위 |
| --- | --- | --- |
| gateway | 모든 public service | 공개 route prefix |
| payment-service | order-service | 필요한 경우에만 event path 또는 internal callback |
| order-service | payment-service | synchronous payment를 사용할 경우 payment API |
| notification-service | order-service | direct write 금지 |
| synthetic runner | public gateway | synthetic header만 허용 |

## 5. Network policy

기준:

- namespace별 default deny ingress
- gateway에서 application service로의 접근 허용
- 필요한 경우에만 app에서 DB/Kafka 접근 허용
- Prometheus가 `/metrics`를 scraping하도록 허용
- OTEL collector 또는 log collector 경로 허용

## 6. Secret 관리

Secret:

- JWT signing key
- DB passwords
- 활성화 시 Kafka credential
- Grafana admin credential
- synthetic test credential

규칙:

- Git에 plaintext secret 금지
- screenshot 또는 evidence에 secret 금지
- log에 secret 금지
- 문서화된 절차로 JWT signing key rotate

## 7. 입력과 남용 제어

주문 경로 제어:

- `Idempotency-Key` 요구
- `quantity` 검증
- open time 전 주문 거절
- admission rate limit
- request body size limit
- 가능한 경우 사용자별, IP별 throttle 적용

Catalog 제어:

- pagination 제한
- unbounded search query 금지
- 공개 응답만 cache

## 8. Audit logging

Audit event:

- operator가 drop 생성 또는 수정
- operator가 quantity 또는 open time 변경
- operator가 drop 취소
- admin이 DLQ 조회 또는 event replay
- 수동 rollback 실행

필드:

- actor id
- actor role
- action
- target type
- target id
- request id
- trace id
- result
- timestamp

password, JWT, payment secret은 포함하지 않는다.

## 9. 보안 테스트 체크리스트

- [ ] customer는 다른 customer의 order를 읽을 수 없다.
- [ ] customer는 admin drop API를 호출할 수 없다.
- [ ] JWT가 없으면 401을 반환한다.
- [ ] role이 맞지 않으면 403을 반환한다.
- [ ] 다른 payload로 idempotency key를 재사용하면 409를 반환한다.
- [ ] internal admin endpoint는 외부 route로 접근할 수 없다.
- [ ] notification outage가 internal error body를 노출하지 않는다.
- [ ] log에 Authorization header가 포함되지 않는다.
- [ ] metric label에 customer id 또는 order id가 포함되지 않는다.
