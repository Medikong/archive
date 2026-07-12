# Global Companies

해외 기업의 인증 서비스 설계 사례를 회사별로 정리한다.

## 회사 인덱스

| 회사 | 조사 상태 | 주요 주제 | 시작 문서 |
| --- | --- | --- | --- |
| Google | 분석 | OIDC ID token 검증, OAuth refresh token, IAP, security challenge | [google/README.md](google/README.md) |
| Microsoft | 분석 | Entra ID, Continuous Access Evaluation, risk-based Conditional Access, MFA | [microsoft/README.md](microsoft/README.md) |
| Netflix | 분석 | edge authentication, Passport identity, device-bound identity, suspicious connection MFA | [netflix/README.md](netflix/README.md) |
| Uber | 분석 | SPIFFE/SPIRE, workload identity, ABAC, policy distribution | [uber/README.md](uber/README.md) |
| Stripe | 분석 | restricted API keys, IP restriction, key expiry, least privilege | [stripe/README.md](stripe/README.md) |
| GitHub | 분석 | GitHub Apps, OAuth Apps, fine-grained PAT, short-lived tokens | [github/README.md](github/README.md) |
| Auth0 | 분석 | account linking, refresh token rotation, reuse detection, Adaptive MFA | [auth0/README.md](auth0/README.md) |

## 이번 조사에서 얻은 해외 사례 포인트

- [Google](https://developers.google.com/identity/openid-connect/openid-connect), [LINE](https://developers.line.biz/en/docs/line-login/verify-id-token/), [Kakao](https://developers.kakao.com/docs/en/kakaologin/common)처럼 OIDC를 제공하는 provider는 ID token을 발급하지만, 서비스는 issuer, audience, signature, expiry를 직접 검증해야 한다.
- [Microsoft](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)와 [Auth0](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)는 단순 토큰 만료시간보다 정책 재평가, refresh token rotation, reuse detection, 위험 기반 MFA를 운영 중심으로 다룬다.
- [Netflix](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)와 [Uber](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/)는 대규모 내부 서비스에서 사용자 토큰을 모든 서비스가 직접 해석하게 두지 않고, edge/gateway나 공통 인증 라이브러리로 identity를 정규화한다.
- [GitHub](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps)와 [Stripe](https://docs.stripe.com/keys/restricted-api-keys)는 user-facing 로그인보다 개발자/API 인증 사례에 가깝지만, 권한 범위 축소, 만료, 조직 승인, IP 제한 같은 운영 원칙을 참고할 수 있다.

## 회사별 폴더 규칙

```text
{company}/
  README.md
  posts/
```

- 회사 폴더명은 소문자 kebab-case로 쓴다.
- `README.md`는 회사별 요약과 주요 설계 포인트를 담는다.
- `posts/`에는 포스트별 스크랩을 둔다.
- `assets/`는 원문 도식, 캡처, 참고 이미지가 필요할 때만 둔다.
