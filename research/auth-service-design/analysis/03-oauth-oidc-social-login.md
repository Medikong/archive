# OAuth, OIDC, 소셜 로그인

## 질문

외부 IdP와 소셜 로그인은 내부 계정과 어떻게 연결되는가?

## 원문에서 확인한 내용

- [Kakao Login REST API는 authorization code를 token으로 교환하고, 서비스 서버가 access token으로 사용자 정보를 조회해 서비스 회원 여부를 판단한다고 설명한다](https://developers.kakao.com/docs/en/kakaologin/rest-api).
- [Kakao Concepts는 OIDC 활성화 시 ID token을 추가 발급한다고 설명한다](https://developers.kakao.com/docs/en/kakaologin/common).
- [LINE Login v2.1은 OAuth 2.0 authorization code grant와 OIDC를 지원하며](https://developers.line.biz/en/docs/line-login/integrate-line-login/), [ID token claim과 서명 검증 방식도 별도 문서로 설명한다](https://developers.line.biz/en/docs/line-login/verify-id-token/).
- [Google OIDC 문서는 ID token을 신뢰하기 전에 signature, issuer, audience, expiry를 검증해야 한다고 설명한다](https://developers.google.com/identity/openid-connect/openid-connect).
- [OpenID Connect Core는 OIDC가 OAuth 2.0 위에 identity layer를 추가하고, ID token을 통해 인증 정보를 전달한다고 설명한다](https://openid.net/specs/openid-connect-core-1_0.html).

## OAuth/OIDC 경계

| 항목 | OAuth 2.0 | OIDC |
| --- | --- | --- |
| 핵심 목적 | resource 접근 권한 위임 | 사용자 인증 결과 전달 |
| 주요 산출물 | access token, refresh token | ID token, UserInfo claim |
| DropMong 사용처 | provider API 호출, social profile 조회 | 외부 사용자 인증 확인 |
| 주의점 | access token을 내부 사용자 id로 쓰지 않는다. | ID token은 서버 검증 전까지 신뢰하지 않는다. |

## 소셜 로그인 처리 과정

1. 클라이언트가 `GET /auth/oauth/{provider}/start`를 호출한다.
2. `auth-service`가 `state`, `nonce`, `redirect_uri`, `provider`를 저장하고 provider authorization endpoint로 redirect한다.
3. provider가 사용자 인증과 동의를 처리한 뒤 callback으로 `code`, `state`를 전달한다.
4. `auth-service`가 `state`를 검증하고 `code`를 token으로 교환한다.
5. OIDC provider라면 ID token signature, issuer, audience, expiry, nonce를 검증한다.
6. provider subject를 `external_identity(provider, subject)`에서 찾는다.
7. 연결된 내부 `member_id`가 없으면 가입/연결 후보 상태를 만든다.
8. `member-service`가 회원 상태, 약관 동의, 정지/탈퇴 여부를 판단한다.
9. 내부 session, access token, refresh token을 발급한다.

## DB 적용안

### `oauth_login_attempts`

| 필드 | 설명 |
| --- | --- |
| `id` | login attempt id |
| `provider` | kakao, naver, line, google 등 |
| `state_hash` | 원문 state가 아닌 hash |
| `nonce_hash` | 원문 nonce가 아닌 hash |
| `redirect_uri` | 등록된 redirect URI |
| `status` | pending, consumed, expired, failed |
| `expires_at`, `consumed_at` | 만료/사용 시각 |

### `external_identities`

| 필드 | 설명 |
| --- | --- |
| `id` | external identity id |
| `member_id` | 내부 회원 id |
| `provider` | provider 이름 |
| `provider_subject` | provider의 stable subject |
| `email` | provider claim에서 받은 email, 유일성 가정 금지 |
| `email_verified` | provider claim 기준 |
| `linked_at`, `revoked_at` | 연결/해제 시각 |

## 확인 필요

- provider별 `sub` 안정성, email 변경 정책, 탈퇴 후 재가입 시 subject 재사용 여부는 provider 문서를 더 확인해야 한다.
- PKCE는 mobile/SPA를 지원할 때 필수에 가깝지만, DropMong 초기 클라이언트 구성이 확정되어야 적용 범위를 정할 수 있다.

## 출처

| 회사 | 출처 | 저장 위치 | 확인한 내용 |
| --- | --- | --- | --- |
| Kakao | https://developers.kakao.com/docs/en/kakaologin/rest-api | companies/domestic/kakao/posts/undated-kakao-login-oauth-oidc.md | authorization code, token, service user id, 내부 회원 판단 |
| LINE | https://developers.line.biz/en/docs/line-login/integrate-line-login/ | companies/domestic/line/posts/undated-line-login-oidc-id-token.md | OAuth 2.0 authorization code grant와 OIDC |
| LINE | https://developers.line.biz/en/docs/line-login/verify-id-token/ | companies/domestic/line/posts/undated-line-login-oidc-id-token.md | ID token claim과 signature validation |
| Google | https://developers.google.com/identity/openid-connect/openid-connect | companies/global/google/posts/undated-google-openid-connect-iap.md | ID token signature, issuer, audience, expiry 검증 |
| OpenID Foundation | https://openid.net/specs/openid-connect-core-1_0.html | sources.md | OIDC가 OAuth 2.0 위에 identity layer를 추가한다는 표준 정의 |
