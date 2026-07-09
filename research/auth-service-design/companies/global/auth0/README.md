# Auth0

Auth0 자료는 [계정 연결](https://auth0.com/docs/manage-users/user-accounts/user-account-linking), [refresh token rotation과 token reuse detection](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation), Adaptive MFA 설계의 직접 참고 자료로 조사했다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| User Account Linking | https://auth0.com/docs/manage-users/user-accounts/user-account-linking | posts/undated-auth0-token-rotation-account-linking-mfa.md |
| Refresh Token Rotation | https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation | posts/undated-auth0-token-rotation-account-linking-mfa.md |
| Configure Refresh Token Rotation | https://auth0.com/docs/secure/tokens/refresh-tokens/configure-refresh-token-rotation | posts/undated-auth0-token-rotation-account-linking-mfa.md |

## 원문 확인 내용

- [Auth0 Account Linking 문서](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)는 여러 identity provider의 계정을 하나의 user profile에 연결할 수 있지만, 기본적으로 identity를 별도 사용자로 취급한다고 설명한다.
- [Auth0 Account Linking 문서](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)는 account linking에서 primary account와 secondary account가 있고, secondary identity가 primary profile의 identities 배열에 들어간다고 설명한다.
- [Auth0 Account Linking 문서](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)는 안전하지 않은 account linking이 계정 탈취로 이어질 수 있으므로, manual/automatic linking 모두 양쪽 계정 인증을 요구해야 한다고 설명한다.
- [Auth0 Refresh Token Rotation 문서](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)는 refresh token을 교환할 때마다 새 refresh token을 발급하고 이전 token을 폐기한다고 설명한다.
- [Auth0 Refresh Token Rotation 문서](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)는 이미 무효화된 refresh token이 재사용되면 token family를 무효화하고 재인증을 요구한다고 설명한다.
- [Auth0 Configure Refresh Token Rotation 문서](https://auth0.com/docs/secure/tokens/refresh-tokens/configure-refresh-token-rotation)는 rotation overlap/leeway가 네트워크 재시도나 동시 요청 때문에 정상 사용자가 로그아웃되는 상황을 줄이기 위한 설정이라고 설명한다.

## 설계 포인트

- 계정 연결은 email 동일성만으로 자동 처리하면 위험하다.
- refresh token은 단일 row가 아니라 family와 rotation history로 보아야 한다.
- replay 탐지와 정상 동시성 재시도 사이의 균형이 필요하다.

## DropMong 적용 아이디어

- `external_identity`는 primary `member_id` 아래 여러 provider identity를 가질 수 있지만, 연결 전 재인증을 요구한다.
- `refresh_token_family`와 `refresh_token`을 나누고, 재사용 탐지 시 family 전체를 폐기한다.
- refresh token rotation은 짧은 leeway를 두되, leeway 내 허용된 재시도도 audit event로 남긴다.

## 확인 필요

- [Auth0 공개 문서](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)에서는 Adaptive MFA 세부 risk score 산정 방식을 확인할 수 없다.
