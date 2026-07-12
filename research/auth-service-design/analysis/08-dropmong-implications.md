# DropMong 적용 메모

## 질문

국내/해외 기업 사례에서 DropMong 인증/회원 설계에 적용할 점은 무엇인가?

## 바로 적용할 설계 원칙

1. 인증 서비스와 회원 서비스를 분리한다.
2. 외부 provider subject와 내부 `member_id`를 직접 동일시하지 않는다.
3. access token은 짧게, refresh token은 [rotation과 family revocation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)을 둔다.
4. social account linking은 [동일 email만으로 자동 병합하지 않는다](https://openid.net/specs/openid-connect-core-1_0.html).
5. 민감 작업은 별도 `auth_challenge` transaction으로 [step-up 인증](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies)을 요구한다.
6. 내부 서비스에는 외부 provider token이 아니라 [검증된 identity context](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)를 전달한다.
7. 로그인, token, MFA, 계정 연결, 권한 변경은 [감사 로그와 보안 이벤트](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)로 남긴다.

## 서비스 책임 경계

| 서비스 | 책임 | 책임이 아닌 것 |
| --- | --- | --- |
| `auth-service` | credential 검증, social/OIDC callback, session/token 발급, refresh token rotation, MFA challenge, account linking 검증, auth audit event | 회원 프로필 관리, 마케팅 동의, 배송지, 구매 이력 |
| `member-service` | 회원 상태, 프로필, 약관/동의, 휴면/탈퇴/정지 상태, 회원 조회 API | password hash, provider token, refresh token 원문 |
| `gateway` | 외부 access token 검증, 내부 identity context 생성/전달, rate limit 일부 | 회원 상태의 최종 변경, provider callback 처리 |
| domain services | 인증된 `member_id`와 권한 context를 받아 도메인 규칙 수행 | 외부 OAuth token 검증 |

## 우선 DB 후보

| 테이블 | 핵심 필드 | 우선순위 |
| --- | --- | --- |
| `members` | `id`, `status`, `created_at`, `deleted_at` | P0 |
| `member_profiles` | `member_id`, `display_name`, `phone`, `email` | P0 |
| `credentials` | `member_id`, `type`, `password_hash`, `created_at`, `revoked_at` | P0 |
| `external_identities` | `member_id`, `provider`, `provider_subject`, `linked_at`, `revoked_at` | P0 |
| `sessions` | `member_id`, `device_id`, `status`, `auth_time`, `assurance_level`, `revoked_at` | P0 |
| `refresh_token_families` | `session_id`, `status`, `reuse_detected_at`, `revoked_at` | P0 |
| `refresh_tokens` | `family_id`, `token_hash`, `previous_token_id`, `expires_at`, `rotated_at`, `revoked_at` | P0 |
| `oauth_login_attempts` | `provider`, `state_hash`, `nonce_hash`, `status`, `expires_at` | P0 |
| `auth_challenges` | `member_id`, `session_id`, `challenge_type`, `reason`, `status`, `expires_at` | P1 |
| `auth_audit_events` | `actor_type`, `actor_id`, `event_type`, `ip_hash`, `user_agent_hash`, `metadata` | P0 |
| `api_keys` | `owner_type`, `owner_id`, `key_hash`, `scopes`, `expires_at`, `revoked_at` | P2 |

## 우선 API 후보

| API | 책임 | 우선순위 |
| --- | --- | --- |
| `POST /auth/login` | password 로그인 | P0 |
| `GET /auth/oauth/{provider}/start` | social/OIDC 시작, state/nonce 저장 | P0 |
| `GET /auth/oauth/{provider}/callback` | code 교환, ID token 검증, 내부 로그인 처리 | P0 |
| `POST /auth/token/refresh` | refresh token rotation | P0 |
| `POST /auth/logout` | 현재 session 폐기 | P0 |
| `POST /auth/logout-all` | 모든 session 폐기 | P1 |
| `GET /auth/sessions` | 기기/session 목록 | P1 |
| `DELETE /auth/sessions/{session_id}` | 특정 session 폐기 | P1 |
| `POST /auth/challenges` | 민감 작업 추가 인증 시작 | P1 |
| `POST /auth/social-connections/{provider}/start` | 계정 연결용 provider 재인증 | P1 |
| `DELETE /auth/social-connections/{provider}` | provider 연결 해제와 token revocation | P1 |

## 현재 범위에서는 보류할 점

- [Netflix Passport](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602) 수준의 독립 identity propagation platform은 초기 DropMong에는 과하다. gateway-signed identity context와 공통 middleware로 시작한다.
- [Uber SPIFFE/SPIRE](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/)는 장기적으로 좋지만, 초기에는 Kubernetes service account, network policy, gateway 서명 token으로 출발한다.
- [Microsoft Entra](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies) 수준의 ML risk engine은 초기 범위에 넣지 않는다. 로그인 실패 횟수, 새 기기, IP 급변, 고가 구매 같은 rule 기반 신호부터 사용한다.
- [Stripe](https://docs.stripe.com/keys/restricted-api-keys)/[GitHub](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)식 partner key 관리 UI는 외부 연동 API가 생긴 뒤 P2로 둔다.

## 바로 남길 ADR 후보

- `auth-service`와 `member-service` 책임 분리
- refresh token rotation과 family revocation 도입
- social login provider subject와 내부 member id 분리
- gateway internal identity context 형식
- auth audit event 보존 범위와 마스킹 정책

## 적용 후보

| 주제 | 참고 사례 | 적용 방향 | 보류할 점 |
| --- | --- | --- | --- |
| 외부 로그인 | [Kakao](https://developers.kakao.com/docs/en/kakaologin/rest-api), [Naver](https://developers.naver.com/docs/login/devguide/devguide.md), [LINE](https://developers.line.biz/en/docs/line-login/integrate-line-login/), [Google](https://developers.google.com/identity/openid-connect/openid-connect) | provider 인증 후 내부 member 상태 판단 | provider 내부 계정 정책 단정 금지 |
| refresh token | [Kakao](https://developers.kakao.com/docs/en/kakaologin/common), [Auth0](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation), [Naver](https://developers.naver.com/docs/login/devguide/devguide.md) | rotation, family revocation, token hash 저장 | sender-constrained token은 장기 검토 |
| 계정 연결 | [Auth0](https://auth0.com/docs/manage-users/user-accounts/user-account-linking), [Naver](https://developers.naver.com/docs/login/devguide/devguide.md) | 양쪽 계정 재인증, 연결/해제 감사 로그 | 동일 email 자동 병합 |
| MFA/위험 인증 | [Toss](https://toss.tech/article/slash23-security), [Google](https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges), [Microsoft](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies), [Netflix](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602) | 민감 작업 step-up challenge | 모든 로그인 MFA 강제 |
| 내부 서비스 인증 | [Netflix](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602), [Uber](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/), [Google IAP](https://docs.cloud.google.com/iap/docs/concepts-overview) | gateway-signed identity context, 공통 middleware | 독립 identity platform 초기 도입 |
| API key 운영 | [Stripe](https://docs.stripe.com/keys/restricted-api-keys), [GitHub](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) | scope, 만료, 승인, IP 제한 | partner API 전용 UI 즉시 구현 |
