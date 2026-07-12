# Auth0 Token Rotation and Account Linking

- 회사: Auth0
- 원문: https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- 원문: https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation
- 원문: https://auth0.com/docs/secure/tokens/refresh-tokens/configure-refresh-token-rotation
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: account linking, refresh token rotation, reuse detection, leeway

## 원문 확인 내용

[Auth0 Account Linking 문서](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)는 서로 다른 identity provider의 계정을 하나의 user profile에 연결할 수 있다고 설명한다. 다만 기본적으로는 서로 다른 identity를 별도 사용자로 취급하며, 연결 시 primary account와 secondary account를 명시한다.

[Auth0 Account Linking 문서](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)는 안전하지 않은 연결이 계정 탈취를 만들 수 있으므로, 자동 연결과 수동 연결 모두에서 양쪽 계정 인증이 필요하다고 설명한다. 동일 email은 연결 제안 근거가 될 수 있지만 단독 확정 조건으로 쓰기 어렵다.

[Auth0 Refresh Token Rotation 문서](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)는 refresh token 교환 때마다 새 refresh token을 발급하고 이전 token을 폐기한다고 설명한다. 이미 폐기된 token이 재사용되면 token family 전체를 무효화해 재인증을 요구한다. [Auth0 Configure Refresh Token Rotation 문서](https://auth0.com/docs/secure/tokens/refresh-tokens/configure-refresh-token-rotation)는 네트워크 재시도와 동시성 문제를 줄이기 위한 leeway를 설명한다.

## 적용 아이디어

- DropMong account linking API는 현재 session만으로 처리하지 않고, 연결하려는 provider에 대한 재인증을 요구한다.
- refresh token은 `family_id`, `previous_token_id`, `rotated_at`, `revoked_at`, `reuse_detected_at`을 기록한다.
- rotation leeway는 짧게 두고, 같은 token 재사용이 leeway 밖에서 발생하면 family 전체를 폐기한다.

## 확인 필요

- [Auth0 공개 문서](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)에서는 정확한 Adaptive MFA risk score 계산 방법을 확인할 수 없다.
