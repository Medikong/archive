# 내부 서비스 인증

## 질문

내부 서비스 간 인증은 어떤 방식으로 설계되는가?

## 원문에서 확인한 내용

- [Netflix는 edge gateway와 Edge Authentication Services에서 인증/identity 처리를 중앙화하고, downstream 서비스에는 Passport라는 공통 identity 구조를 전달한다고 설명한다](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602).
- [Google IAP는 application 앞에 중앙 authorization 계층을 두어 application-level 접근제어를 제공한다고 설명한다](https://docs.cloud.google.com/iap/docs/concepts-overview).
- [Uber는 SPIFFE/SPIRE와 공통 Auth library로 workload identity와 service-to-service 인증을 추상화한다고 설명한다](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/).
- [Uber ABAC 글은 actor, action, resource를 구분하고 중앙 policy service에서 정책을 관리/배포한다고 설명한다](https://www.uber.com/us/en/blog/attribute-based-access-control-at-uber/).
- [GitHub는 user OAuth app과 app/installation 주체를 구분하고 short-lived token과 fine-grained permission을 강조한다](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps).
- [Stripe는 API key 권한을 restricted key로 좁히고 IP 제한을 둘 수 있다고 설명한다](https://docs.stripe.com/keys/restricted-api-keys).

## 사용자 인증과 서비스 인증 분리

| 구분 | 목적 | 예시 | DropMong 적용 |
| --- | --- | --- | --- |
| user authentication | 고객이 누구인지 확인 | password, social login, OIDC ID token | `member_id`, `session_id` 발급 |
| user authorization | 고객이 무엇을 할 수 있는지 판단 | 구매, 쿠폰 사용, 계정 변경 | member 상태, policy, ownership 확인 |
| service authentication | 호출한 workload/service가 무엇인지 확인 | mTLS, SPIFFE ID, service token | `service_id`, workload identity |
| service authorization | 서비스가 어떤 내부 API를 호출할 수 있는지 판단 | order-service -> payment-service | internal scope, policy |

## 내부 identity context

초기 DropMong에서는 [Netflix Passport](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)처럼 별도 대형 시스템을 만들기보다, gateway가 서명한 내부 identity context로 시작할 수 있다.

| claim | 설명 |
| --- | --- |
| `member_id` | 인증된 내부 회원 |
| `session_id` | 현재 session |
| `device_id` | 기기/session 단위 식별자 |
| `auth_time` | 최초 인증 시각 |
| `assurance_level` | password/social/mfa 등 인증 강도 |
| `risk_level` | low/medium/high |
| `actor_type` | user 또는 service |
| `issuer` | gateway 또는 auth-service |
| `issued_at`, `expires_at` | 내부 context 수명 |

## 내부 서비스 호출 방식

| 단계 | 권장 방식 |
| --- | --- |
| 초기 | gateway에서 외부 token 검증, 내부 서비스에는 짧은 수명의 signed identity context 전달 |
| 확장 | 서비스별 auth middleware로 context signature, audience, expiry 검증 |
| 운영 안정화 | service-to-service token에 issuer/audience/scope를 두고 감사 로그 연결 |
| 장기 | service mesh mTLS 또는 SPIFFE/SPIRE 기반 workload identity 검토 |

## 확인 필요

- DropMong 배포 환경에서 Istio/service mesh를 사용할지, Kubernetes service account와 gateway 서명 token으로 시작할지 결정해야 한다.
- 내부 context를 HTTP header로 전달한다면 gateway 밖에서 header 위조가 불가능하도록 mTLS, network policy, signature 검증이 필요하다.

## 출처

| 회사 | 출처 | 저장 위치 | 확인한 내용 |
| --- | --- | --- | --- |
| Netflix | https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602 | companies/global/netflix/posts/undated-netflix-edge-authentication-passport.md | edge authentication과 Passport identity propagation |
| Google | https://docs.cloud.google.com/iap/docs/concepts-overview | companies/global/google/posts/undated-google-openid-connect-iap.md | 중앙 authorization layer |
| Uber | https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/ | companies/global/uber/posts/undated-uber-spiffe-spire-abac.md | SPIFFE/SPIRE와 workload identity |
| Uber | https://www.uber.com/us/en/blog/attribute-based-access-control-at-uber/ | companies/global/uber/posts/undated-uber-spiffe-spire-abac.md | actor/action/resource와 policy distribution |
| GitHub | https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps | companies/global/github/posts/undated-github-apps-oauth-fine-grained-token.md | user token과 app/installation token의 분리 |
| Stripe | https://docs.stripe.com/keys/restricted-api-keys | companies/global/stripe/posts/undated-stripe-restricted-api-keys.md | restricted API key와 최소 권한 |
