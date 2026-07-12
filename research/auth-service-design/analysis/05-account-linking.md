# 계정 연결

## 질문

여러 인증 수단과 계정 병합은 어떻게 다루는가?

## 원문에서 확인한 내용

- [Auth0는 서로 다른 identity provider의 계정을 하나의 user profile에 연결할 수 있지만, 기본적으로는 서로 다른 identity를 별도 사용자로 취급한다고 설명한다](https://auth0.com/docs/manage-users/user-accounts/user-account-linking).
- [Auth0 account linking은 primary account와 secondary account를 지정하고, secondary account의 identity가 primary profile에 포함되는 구조를 설명한다](https://auth0.com/docs/manage-users/user-accounts/user-account-linking).
- [Auth0는 안전하지 않은 account linking이 계정 탈취를 만들 수 있으므로, 자동/수동 연결 모두 양쪽 계정 인증이 필요하다고 설명한다](https://auth0.com/docs/manage-users/user-accounts/user-account-linking).
- [Naver는 연동 해제가 provider 연결 관계를 끊고 access/refresh token을 폐기하는 작업이라고 설명한다](https://developers.naver.com/docs/login/devguide/devguide.md).
- [OIDC Core는 `email` claim의 유일성에 의존하면 안 된다고 설명한다](https://openid.net/specs/openid-connect-core-1_0.html).

## 계정 연결 원칙

| 원칙 | 이유 |
| --- | --- |
| provider subject를 내부 member id로 쓰지 않는다 | provider별 subject 안정성과 범위가 다르다. |
| 동일 email만으로 자동 병합하지 않는다 | email 재사용, 미검증 email, provider별 검증 정책 차이가 있다. |
| 연결 전 양쪽 계정 재인증을 요구한다 | 현재 session 탈취 상태에서 다른 계정을 연결하는 공격을 막는다. |
| primary/secondary 개념을 둔다 | 병합 후 남는 내부 회원과 사라지는 후보를 명확히 해야 한다. |
| 연결 해제와 탈퇴를 분리한다 | provider 연결 해제는 내부 회원 탈퇴와 다르다. |

## DropMong 모델

### `external_identities`

| 필드 | 설명 |
| --- | --- |
| `member_id` | primary 내부 회원 |
| `provider` | kakao, naver, line, google 등 |
| `provider_subject` | provider stable id |
| `linked_by_session_id` | 연결을 수행한 session |
| `linked_challenge_id` | 연결 전 재인증 challenge |
| `status` | active, revoked |
| `revoked_at`, `revoked_reason` | 연결 해제 정보 |

### `account_link_requests`

| 필드 | 설명 |
| --- | --- |
| `id` | 연결 요청 id |
| `primary_member_id` | 유지할 내부 회원 |
| `provider` | 연결하려는 provider |
| `provider_subject` | 연결하려는 provider subject |
| `status` | pending, verified, linked, rejected, expired |
| `created_at`, `expires_at`, `linked_at` | 요청 시각 |

## API 적용안

| API | 책임 |
| --- | --- |
| `GET /auth/social-connections` | 현재 연결된 provider 목록 조회 |
| `POST /auth/social-connections/{provider}/start` | 연결용 OAuth/OIDC 재인증 시작 |
| `POST /auth/social-connections/{provider}/callback` | provider callback 검증 후 연결 후보 생성 |
| `POST /auth/social-connections/{provider}/confirm` | 현재 계정 재인증 뒤 연결 확정 |
| `DELETE /auth/social-connections/{provider}` | provider 연결 해제와 token revocation |

## 확인 필요

- DropMong에서 guest 계정이나 임시 구매 시도를 허용할 경우, guest-to-member 전환 정책을 별도로 설계해야 한다.
- 동일 전화번호를 여러 계정에서 사용할 수 있는지, 탈퇴 후 재가입 시 provider identity를 재사용할 수 있는지는 제품 정책이 필요하다.

## 출처

| 회사 | 출처 | 저장 위치 | 확인한 내용 |
| --- | --- | --- | --- |
| Auth0 | https://auth0.com/docs/manage-users/user-accounts/user-account-linking | companies/global/auth0/posts/undated-auth0-token-rotation-account-linking-mfa.md | primary/secondary account와 양쪽 계정 인증 필요 |
| Naver | https://developers.naver.com/docs/login/devguide/devguide.md | companies/domestic/naver/posts/undated-naver-login-token-revocation.md | provider 연동 해제와 token revocation |
| Kakao | https://developers.kakao.com/docs/en/kakaologin/rest-api | companies/domestic/kakao/posts/undated-kakao-login-oauth-oidc.md | service user id 기반 내부 회원 판단 |
| LINE | https://developers.line.biz/en/docs/line-login/verify-id-token/ | companies/domestic/line/posts/undated-line-login-oidc-id-token.md | OIDC `sub`, `email`, `email_verified` claim |
| OpenID Foundation | https://openid.net/specs/openid-connect-core-1_0.html | sources.md | email claim의 유일성에 의존하지 말라는 표준 주의사항 |
