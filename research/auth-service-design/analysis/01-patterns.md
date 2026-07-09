# 인증 서비스 반복 패턴

## 질문

여러 회사에서 반복되는 인증 서비스 구조는 무엇인가?

## 원문에서 확인한 반복 구조

### 1. 외부 인증과 내부 회원 판단은 분리된다

[Kakao Login REST API](https://developers.kakao.com/docs/en/kakaologin/rest-api)와 [Naver Login 개발 가이드](https://developers.naver.com/docs/login/devguide/devguide.md)는 외부 provider가 사용자 인증과 동의를 처리하더라도, 서비스 서버가 provider token으로 사용자 정보를 조회한 뒤 내부 서비스 회원인지 판단해야 한다고 설명한다. [LINE ID token 검증 문서](https://developers.line.biz/en/docs/line-login/verify-id-token/)와 [Google OIDC 문서](https://developers.google.com/identity/openid-connect/openid-connect)도 ID token을 받아도 서비스가 서명, issuer, audience, expiry를 검증해야 한다고 설명한다.

따라서 provider 인증 결과는 내부 `member` 자체가 아니라 내부 회원을 찾기 위한 증거로 보는 편이 안전하다.

### 2. credential, external identity, member, profile은 다른 모델이다

[Auth0 account linking 문서](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)는 여러 provider identity를 하나의 primary user profile에 연결할 수 있지만, 기본적으로 서로 다른 identity를 별도 사용자로 취급한다고 설명한다. 같은 문서는 primary account와 secondary account를 구분하고, profile metadata 자동 병합을 하지 않는다고 설명한다.

DropMong 관점에서는 다음처럼 나누는 것이 적합하다.

| 개념 | 책임 | 예시 |
| --- | --- | --- |
| credential | 사용자가 제시하는 증명 수단 | password hash, passkey credential, social provider token |
| external identity | 외부 provider의 사용자 식별자 | Kakao service user id, LINE `sub`, Google `sub` |
| member/account | DropMong 내부의 고객 계정 | `member_id`, 상태, 가입일, 탈퇴/정지 상태 |
| user profile | 서비스 화면과 커뮤니케이션에 쓰는 정보 | 이름, 닉네임, 전화번호, 배송지 일부 |

### 3. token/session은 만료시간뿐 아니라 폐기와 재사용 탐지가 필요하다

[Kakao는 refresh token 갱신 시 새 refresh token을 발급하고 이전 refresh token을 폐기한다고 설명한다](https://developers.kakao.com/docs/en/kakaologin/common). [Naver는 token revocation에서 access token과 refresh token 쌍을 함께 폐기하는 방식을 설명한다](https://developers.naver.com/docs/login/devguide/devguide.md). [Auth0는 refresh token rotation과 reuse detection을 설명하며](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation), 이미 무효화된 token이 재사용되면 token family 전체를 무효화한다.

### 4. gateway/edge 계층에서 identity를 정규화한다

[Netflix는 edge gateway인 Zuul과 Passport 구조로 외부 token 종류에 무관한 identity propagation을 설명한다](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602). [Google IAP는 application 앞에 중앙 authorization 계층을 두는 패턴을 제공한다](https://docs.cloud.google.com/iap/docs/concepts-overview). [Uber는 공통 Auth library와 SPIFFE/SPIRE 기반 workload identity를 통해 service-to-service 인증을 추상화한다](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/).

### 5. MFA와 위험 기반 인증은 모든 요청이 아니라 중요한 순간에 쓰인다

[Microsoft Entra는 sign-in risk와 user risk에 따라 MFA나 차단을 적용할 수 있다고 설명한다](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies). [Google Workspace는 의심 로그인과 민감 작업에 security challenge를 둔다](https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges). [Netflix는 의심스러운 연결에 MFA를 선택적으로 도입하는 방향을 설명한다](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602). [Toss는 내부 업무 시스템 접근에서 IAM, SSO, device 보안 수준, 생체 인증을 함께 확인한다고 설명한다](https://toss.tech/article/slash23-security).

### 6. 운영 보안은 제품 기능으로 다뤄야 한다

[Stripe](https://docs.stripe.com/keys/restricted-api-keys)와 [GitHub](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) 자료는 API key/token의 권한 범위, 만료, 조직 승인, IP 제한 같은 운영 기능을 강조한다. [Auth0는 refresh token reuse detection 이벤트를 로그와 연결할 수 있다고 설명한다](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation). 인증 서비스는 로그인 API만이 아니라 token 폐기, key 회전, 감사 로그, rate limit, 이상 이벤트 탐지를 포함해야 한다.

## DropMong 적용 아이디어

- `auth-service`는 credential, external identity, token/session, MFA challenge, audit event를 책임진다.
- `member-service`는 내부 회원 상태, 프로필, 약관/마케팅 동의, 탈퇴/정지 상태를 책임진다.
- social login은 provider 인증 완료 후 `member-service`의 회원 상태 판단을 거쳐 내부 session을 발급한다.
- refresh token은 단일 token row가 아니라 `refresh_token_family`와 rotation history로 관리한다.
- 내부 서비스에는 외부 provider token을 직접 전달하지 않고 gateway에서 검증된 identity context만 전달한다.

## 출처

| 회사 | 출처 | 저장 위치 | 확인한 내용 |
| --- | --- | --- | --- |
| Kakao | https://developers.kakao.com/docs/en/kakaologin/rest-api | companies/domestic/kakao/posts/undated-kakao-login-oauth-oidc.md | provider 인증 뒤 서비스 서버가 내부 회원 여부 판단 |
| Naver | https://developers.naver.com/docs/login/devguide/devguide.md | companies/domestic/naver/posts/undated-naver-login-token-revocation.md | token revocation과 provider 연동 해제 |
| LINE | https://developers.line.biz/en/docs/line-login/verify-id-token/ | companies/domestic/line/posts/undated-line-login-oidc-id-token.md | ID token claim과 서명 검증 |
| Auth0 | https://auth0.com/docs/manage-users/user-accounts/user-account-linking | companies/global/auth0/posts/undated-auth0-token-rotation-account-linking-mfa.md | primary/secondary identity와 account linking 주의사항 |
| Netflix | https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602 | companies/global/netflix/posts/undated-netflix-edge-authentication-passport.md | edge authentication과 공통 identity 구조 |
| Uber | https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/ | companies/global/uber/posts/undated-uber-spiffe-spire-abac.md | workload identity와 공통 Auth library |
