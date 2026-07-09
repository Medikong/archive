# 운영과 보안

## 질문

키 회전, 감사 로그, rate limit, abuse 대응은 어떻게 운영되는가?

## 원문에서 확인한 내용

- [Auth0 refresh token rotation 문서는 이미 폐기된 refresh token 재사용 시 token family를 무효화하고, reuse detection 이벤트를 로그와 연결할 수 있다고 설명한다](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation).
- [Naver Token Revocation 문서는 access/refresh token 폐기와 provider 연결 해제를 함께 설명한다](https://developers.naver.com/docs/login/devguide/devguide.md).
- [Stripe API key 문서는 restricted key, IP restriction, key expiry를 설명한다](https://docs.stripe.com/keys).
- [GitHub fine-grained PAT 문서는 token expiration, resource owner, repository access, permission, organization approval을 설명한다](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens).
- [Microsoft Entra CAE는 critical event와 정책 평가를 token 접근 재평가에 연결한다](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation).
- [Toss](https://toss.tech/article/slash23-security)와 [Google Workspace](https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges) 자료는 device posture, security challenge, 민감 작업 추가 확인을 운영 정책으로 다룬다.
- [Netflix는 공통 identity 구조가 로그에서 provenance를 추적하고 identity anomaly를 디버깅하는 데 도움이 된다고 설명한다](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602).

## 감사 로그 필수 이벤트

| 이벤트 | 이유 |
| --- | --- |
| login_success, login_failure | credential stuffing, brute force 탐지 |
| token_issued, token_refreshed | token 발급/회전 추적 |
| token_revoked, token_reuse_detected | 탈취나 replay 대응 |
| session_revoked, logout_all | 계정 보호 조치 추적 |
| social_identity_linked, social_identity_unlinked | 계정 연결/해제 감사 |
| mfa_challenge_requested, mfa_challenge_verified, mfa_challenge_failed | 민감 작업 인증 추적 |
| password_changed, credential_added, credential_removed | credential 변경 이력 |
| admin_policy_changed, role_changed | 운영자 권한 변경 감사 |

## rate limit과 abuse 대응

| 대상 | 기준 | 대응 |
| --- | --- | --- |
| 로그인 실패 | member identifier, IP, device fingerprint | 점진적 지연, captcha/challenge, 일시 차단 |
| OAuth callback 실패 | state mismatch, code replay | attempt 폐기, 보안 이벤트 기록 |
| token refresh | refresh token reuse, IP/user-agent 급변 | family 폐기, session 잠금 후보 |
| 본인확인/challenge | txId 반복 조회, 만료 후 재시도 | 조회 횟수 제한, 재발급 요구 |
| API key | IP 제한 위반, scope mismatch | 요청 거부, key owner 알림 |

## key/secret 운영

- access token signing key는 `kid`를 두고 rotation할 수 있어야 한다.
- refresh token 원문은 저장하지 않고 hash만 저장한다.
- API key는 prefix/key id와 secret을 분리하고, DB에는 secret hash만 둔다.
- provider client secret은 secret manager에 보관하고, 문서나 로그에는 남기지 않는다.
- key rotation은 새 key 발급, dual validation, 이전 key 비활성화, 폐기 확인 순서로 runbook화한다.

## 장애와 degraded mode

- auth-service가 완전히 내려가면 신규 로그인과 token refresh는 실패하는 것이 맞다.
- 이미 발급된 access token은 짧은 만료시간 안에서 gateway 검증으로 처리할 수 있다.
- refresh token DB가 불안정할 때 성공처럼 처리하면 계정 탈취 위험이 커진다. 실패를 명확히 반환하고 재시도/안내 정책을 둔다.
- 감사 로그 저장소 장애가 있어도 인증 성공을 무조건 막을지, 임시 buffer를 둘지는 보안 등급별로 결정해야 한다.

## 확인 필요

- DropMong의 개인정보/보안 이벤트 보존 기간, 관리자 열람 권한, 마스킹 정책은 별도 정책 문서가 필요하다.
- 외부 본인확인 provider를 사용할 경우, provider 장애 시 대체 인증 수단이 필요한지 결정해야 한다.

## 출처

| 회사 | 출처 | 저장 위치 | 확인한 내용 |
| --- | --- | --- | --- |
| Auth0 | https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation | companies/global/auth0/posts/undated-auth0-token-rotation-account-linking-mfa.md | token family revocation과 reuse detection logging |
| Naver | https://developers.naver.com/docs/login/devguide/devguide.md | companies/domestic/naver/posts/undated-naver-login-token-revocation.md | token revocation과 provider 연결 해제 |
| Stripe | https://docs.stripe.com/keys | companies/global/stripe/posts/undated-stripe-restricted-api-keys.md | IP restriction과 key expiry |
| GitHub | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | companies/global/github/posts/undated-github-apps-oauth-fine-grained-token.md | fine-grained token, expiration, approval |
| Microsoft | https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation | companies/global/microsoft/posts/undated-microsoft-entra-cae-risk-mfa.md | event-driven access reevaluation |
| Netflix | https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602 | companies/global/netflix/posts/undated-netflix-edge-authentication-passport.md | identity provenance와 anomaly debugging |
