# LINE

[LINE Login integration 문서](https://developers.line.biz/en/docs/line-login/integrate-line-login/)와 [LINE ID token 문서](https://developers.line.biz/en/docs/line-login/verify-id-token/)를 기준으로 OAuth 2.0 authorization code, OIDC, ID token 검증 방식을 확인했다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| Integrating LINE Login with your web app | https://developers.line.biz/en/docs/line-login/integrate-line-login/ | posts/undated-line-login-oidc-id-token.md |
| Get profile information from ID tokens | https://developers.line.biz/en/docs/line-login/verify-id-token/ | posts/undated-line-login-oidc-id-token.md |

## 원문 확인 내용

- [LINE Login v2.1 문서](https://developers.line.biz/en/docs/line-login/integrate-line-login/)는 OAuth 2.0 authorization code grant와 OIDC를 지원한다고 설명한다.
- [LINE Login integration 문서](https://developers.line.biz/en/docs/line-login/integrate-line-login/)는 callback URL에 authorization code와 `state`가 전달된다고 설명한다.
- [LINE ID token 문서](https://developers.line.biz/en/docs/line-login/verify-id-token/)는 ID token이 JWT이고, 서비스가 서명 검증을 해야 한다고 설명한다. web login에서는 channel secret, native/SDK 계열에서는 JWK 기반 검증 조건이 다를 수 있다.
- [LINE ID token 문서](https://developers.line.biz/en/docs/line-login/verify-id-token/)는 ID token payload에 `iss`, `sub`, `aud`, `exp`, `iat`, `nonce`, `amr`, `email` 같은 claim이 포함될 수 있다고 설명한다.

## 설계 포인트

- `sub`는 provider 내부 사용자 식별자이므로 내부 회원 id와 분리한다.
- `email`은 claim으로 받을 수 있지만, OIDC 표준상 email의 유일성에 의존하면 안 된다.
- `state`와 `nonce`는 CSRF, replay 방지 관점에서 로그인 요청 저장소와 함께 설계해야 한다.

## DropMong 적용 아이디어

- `oauth_login_attempt` 테이블에 `state`, `nonce`, `provider`, `redirect_uri`, `expires_at`, `consumed_at`을 둔다.
- ID token 검증 결과는 원문 token 전체가 아니라 필요한 claim과 검증 결과만 감사 로그에 남긴다.
- provider별 `amr` 값은 MFA 판단에 참고하되, 내부 정책의 최종 결정을 대체하지 않는다.

## 확인 필요

- [LINE Login 공개 문서](https://developers.line.biz/en/docs/line-login/verify-id-token/)만으로는 LINE의 내부 계정 연결, refresh token rotation, 이상 로그인 대응을 확인할 수 없다.
