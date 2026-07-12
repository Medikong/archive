# Sources

국내/해외 기업 인증 서비스 설계 조사에 사용한 공개 출처 목록이다.
긴 원문을 복제하지 않고, 원문 링크와 확인 내용 요약을 회사별 스크랩 노트와 분석 문서에 나누어 남긴다.

접근일은 모두 `2026-07-04` 기준이다.

| 상태 | 구분 | 회사 | 제목 | URL | 발행일 | 접근일 | 언어 | 주제 | 저장 위치 | 비고 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 분석 | 국내 | Kakao | Kakao Login Concepts | https://developers.kakao.com/docs/en/kakaologin/common | 미표기 | 2026-07-04 | en | OAuth 2.0, access token, refresh token, OIDC ID token | companies/domestic/kakao/posts/undated-kakao-login-oauth-oidc.md | 토큰 만료와 refresh token 교체 정책 확인 |
| 분석 | 국내 | Kakao | Kakao Login REST API | https://developers.kakao.com/docs/en/kakaologin/rest-api | 미표기 | 2026-07-04 | en | authorization code, consent, service user id, client secret | companies/domestic/kakao/posts/undated-kakao-login-oauth-oidc.md | 서비스 회원 여부 판단을 서비스 서버가 수행 |
| 분석 | 국내 | Naver | 네이버 로그인 API 명세 | https://developers.naver.com/docs/login/api/api.md | 미표기 | 2026-07-04 | ko | OAuth 2.0, authorization code, profile API | companies/domestic/naver/posts/undated-naver-login-token-revocation.md | access token으로 프로필 API 호출 |
| 분석 | 국내 | Naver | 네이버 로그인 개발가이드 | https://developers.naver.com/docs/login/devguide/devguide.md | 미표기 | 2026-07-04 | ko | token revocation, 연동 해제 | companies/domestic/naver/posts/undated-naver-login-token-revocation.md | RFC 7009 기반 토큰 폐기와 연결 해제 확인 |
| 분석 | 국내 | Toss | 금융사 최초의 Zero Trust 아키텍처 도입기 | https://toss.tech/article/slash23-security | 미표기 | 2026-07-04 | ko | IAM, SSO, MFA, device posture, RBAC/ABAC | companies/domestic/toss/posts/undated-toss-zero-trust-iam-sso.md | 내부 업무 시스템 인증 사례 |
| 분석 | 국내 | Toss | 토스 인증 | https://developers-apps-in-toss.toss.im/tossauth/develop.html | 미표기 | 2026-07-04 | ko | 본인확인 API, txId, sessionKey, 서버 간 결과 조회 | companies/domestic/toss/posts/undated-toss-cert-api.md | 사용자 인증 완료와 결과 조회를 분리 |
| 분석 | 국내 | LINE | Integrating LINE Login with your web app | https://developers.line.biz/en/docs/line-login/integrate-line-login/ | 미표기 | 2026-07-04 | en | OAuth 2.0 authorization code, OIDC, callback URL | companies/domestic/line/posts/undated-line-login-oidc-id-token.md | LINE Login v2.1의 OIDC 지원 확인 |
| 분석 | 국내 | LINE | Get profile information from ID tokens | https://developers.line.biz/en/docs/line-login/verify-id-token/ | 미표기 | 2026-07-04 | en | ID token 검증, JWT claim, JWK/channel secret | companies/domestic/line/posts/undated-line-login-oidc-id-token.md | ID token 서명 검증과 `iss`, `sub`, `aud`, `exp` 확인 |
| 분석 | 해외 | Google | OpenID Connect, Sign in with Google | https://developers.google.com/identity/openid-connect/openid-connect | 미표기 | 2026-07-04 | en | ID token 검증, issuer, audience, expiry | companies/global/google/posts/undated-google-openid-connect-iap.md | ID token을 신뢰하기 전 서버 검증 필요 |
| 분석 | 해외 | Google | Using OAuth 2.0 to Access Google APIs | https://developers.google.com/identity/protocols/oauth2 | 미표기 | 2026-07-04 | en | authorization code, refresh token 보관, token limit | companies/global/google/posts/undated-google-openid-connect-iap.md | refresh token 장기 보관과 발급 수 제한 확인 |
| 분석 | 해외 | Google | Identity-Aware Proxy overview | https://docs.cloud.google.com/iap/docs/concepts-overview | 미표기 | 2026-07-04 | en | edge authorization, application-level access control | companies/global/google/posts/undated-google-openid-connect-iap.md | 인증/인가를 중앙 계층에 두는 패턴 |
| 분석 | 해외 | Google | Protect Google Workspace accounts with security challenges | https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges | 미표기 | 2026-07-04 | en | login challenge, sensitive action challenge | companies/global/google/posts/undated-google-openid-connect-iap.md | 위험 상황에서 추가 확인 요구 |
| 분석 | 해외 | Microsoft | Continuous access evaluation in Microsoft Entra | https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation | 미표기 | 2026-07-04 | en | token lifetime, event-driven revocation, CAE | companies/global/microsoft/posts/undated-microsoft-entra-cae-risk-mfa.md | 단순 만료시간보다 이벤트 기반 재평가를 강조 |
| 분석 | 해외 | Microsoft | Microsoft Entra ID Protection risk-based access policies | https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies | 미표기 | 2026-07-04 | en | sign-in risk, user risk, Conditional Access | companies/global/microsoft/posts/undated-microsoft-entra-cae-risk-mfa.md | 위험도에 따라 MFA 또는 차단 적용 |
| 분석 | 해외 | Microsoft | Deployment considerations for Microsoft Entra multifactor authentication | https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-getstarted | 미표기 | 2026-07-04 | en | MFA rollout, Conditional Access, 재인증 빈도 | companies/global/microsoft/posts/undated-microsoft-entra-cae-risk-mfa.md | 과도한 재인증이 피싱 습관을 만들 수 있음 |
| 분석 | 해외 | Netflix | Edge Authentication and Token-Agnostic Identity Propagation | https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602 | 미표기 | 2026-07-04 | en | edge authentication, device-bound identity, Passport, MFA | companies/global/netflix/posts/undated-netflix-edge-authentication-passport.md | downstream에 공통 identity 구조 전달 |
| 분석 | 해외 | Uber | Our Journey Adopting SPIFFE/SPIRE at Scale | https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/ | 미표기 | 2026-07-04 | en | workload identity, SPIFFE/SPIRE, mTLS, auth library | companies/global/uber/posts/undated-uber-spiffe-spire-abac.md | 서비스 간 인증을 개발자 라이브러리와 플랫폼으로 추상화 |
| 분석 | 해외 | Uber | Attribute-Based Access Control at Uber | https://www.uber.com/us/en/blog/attribute-based-access-control-at-uber/ | 미표기 | 2026-07-04 | en | ABAC, actor/action/resource, policy distribution | companies/global/uber/posts/undated-uber-spiffe-spire-abac.md | resource URI와 속성 기반 인가 모델 확인 |
| 분석 | 해외 | GitHub | Differences between GitHub Apps and OAuth apps | https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps | 미표기 | 2026-07-04 | en | OAuth Apps, GitHub Apps, short-lived tokens, permission scope | companies/global/github/posts/undated-github-apps-oauth-fine-grained-token.md | 앱 권한 모델과 사용자 위임 분리 |
| 분석 | 해외 | GitHub | Managing your personal access tokens | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | 미표기 | 2026-07-04 | en | fine-grained PAT, expiration, resource owner, organization approval | companies/global/github/posts/undated-github-apps-oauth-fine-grained-token.md | 토큰 권한을 리소스와 만료 기준으로 제한 |
| 분석 | 해외 | Stripe | API keys | https://docs.stripe.com/keys | 미표기 | 2026-07-04 | en | API key, IP restriction, key expiry | companies/global/stripe/posts/undated-stripe-restricted-api-keys.md | live key IP 제한과 만료 처리 확인 |
| 분석 | 해외 | Stripe | Restricted API keys | https://docs.stripe.com/keys/restricted-api-keys | 미표기 | 2026-07-04 | en | restricted key, least privilege, permission boundary | companies/global/stripe/posts/undated-stripe-restricted-api-keys.md | server-side component별 제한 키 모델 |
| 분석 | 해외 | Auth0 | User Account Linking | https://auth0.com/docs/manage-users/user-accounts/user-account-linking | 미표기 | 2026-07-04 | en | account linking, primary/secondary identity, profile merge | companies/global/auth0/posts/undated-auth0-token-rotation-account-linking-mfa.md | 양쪽 계정 재인증 없이 자동 연결하면 위험 |
| 분석 | 해외 | Auth0 | Refresh Token Rotation | https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation | 미표기 | 2026-07-04 | en | refresh token rotation, reuse detection, token family revocation | companies/global/auth0/posts/undated-auth0-token-rotation-account-linking-mfa.md | replay 탐지 시 token family 무효화 |
| 분석 | 해외 | Auth0 | Configure Refresh Token Rotation | https://auth0.com/docs/secure/tokens/refresh-tokens/configure-refresh-token-rotation | 미표기 | 2026-07-04 | en | rotation overlap, leeway, token lifetime | companies/global/auth0/posts/undated-auth0-token-rotation-account-linking-mfa.md | 네트워크 재시도와 동시성 예외를 leeway로 다룸 |
| 참고 | 표준 | OpenID Foundation | OpenID Connect Core 1.0 incorporating errata set 2 | https://openid.net/specs/openid-connect-core-1_0.html | 2023-12-15 | 2026-07-04 | en | OIDC, ID token, claim, UserInfo | analysis/03-oauth-oidc-social-login.md | 회사 사례 해석을 위한 표준 기준 |

## 상태 값

| 상태 | 의미 |
| --- | --- |
| 후보 | 수집할 가치가 있어 보이나 아직 본문을 확인하지 않음 |
| 수집 | 원문을 스크랩했거나 요약 가능한 수준으로 확인함 |
| 분석 | 회사별 정리나 주제별 분석에 반영함 |
| 제외 | 인증 서비스 설계와 직접 관련이 낮아 제외함 |

## 작성 규칙

- `URL`에는 공개 접근 가능한 원문 링크를 둔다.
- `접근일`은 `YYYY-MM-DD` 형식으로 쓴다.
- `저장 위치`에는 스크랩한 파일 경로를 쓴다.
- 공개 자료에서 확인되지 않은 내용은 비고에 `확인 필요`로 표시한다.
