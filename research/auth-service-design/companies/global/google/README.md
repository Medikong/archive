# Google

Google 자료는 [OIDC ID token 검증](https://developers.google.com/identity/openid-connect/openid-connect), [OAuth refresh token 보관](https://developers.google.com/identity/protocols/oauth2), [Identity-Aware Proxy](https://docs.cloud.google.com/iap/docs/concepts-overview), [Workspace security challenge](https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges) 관점으로 조사했다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| OpenID Connect, Sign in with Google | https://developers.google.com/identity/openid-connect/openid-connect | posts/undated-google-openid-connect-iap.md |
| Using OAuth 2.0 to Access Google APIs | https://developers.google.com/identity/protocols/oauth2 | posts/undated-google-openid-connect-iap.md |
| Identity-Aware Proxy overview | https://docs.cloud.google.com/iap/docs/concepts-overview | posts/undated-google-openid-connect-iap.md |
| Protect Google Workspace accounts with security challenges | https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges | posts/undated-google-openid-connect-iap.md |

## 원문 확인 내용

- [Google OIDC 문서](https://developers.google.com/identity/openid-connect/openid-connect)는 ID token을 신뢰하기 전에 서명, issuer, audience, expiry를 검증해야 한다고 설명한다.
- [Google OAuth 2.0 문서](https://developers.google.com/identity/protocols/oauth2)는 access token 수명이 제한되어 있고, 장기 접근이 필요하면 refresh token을 안전하게 보관해야 한다고 설명한다.
- [Google Cloud IAP 문서](https://docs.cloud.google.com/iap/docs/concepts-overview)는 HTTPS application 앞에 중앙 authorization 계층을 두어 network-level 방화벽 대신 application-level 접근제어를 할 수 있게 한다.
- [Google Workspace security challenge 문서](https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges)는 의심 로그인이나 민감 작업에서 추가 확인을 요구하는 방식을 설명한다.

## 설계 포인트

- ID token은 "사용자라고 주장하는 값"이 아니라 검증을 통과해야 하는 서명된 주장이다.
- refresh token은 장기 보관 대상이므로 암호화, 회전, 폐기, 최대 발급 수, 감사 로그가 함께 필요하다.
- gateway/IAP 계층에서 인증을 끝내고 내부 서비스에는 검증된 identity claim을 전달하는 구조는 microservice에 적합하다.

## DropMong 적용 아이디어

- `auth-service`는 external ID token을 검증한 뒤 내부 access/refresh token을 발급한다.
- gateway는 내부 서비스로 `x-auth-member-id`, `x-auth-session-id`, `x-auth-assurance-level` 같은 검증된 값을 전달하되, 서명 또는 mTLS로 위조를 막는다.
- 민감 작업은 security challenge를 별도 transaction으로 만들고 성공한 challenge만 짧은 시간 재사용한다.

## 확인 필요

- [Google OIDC 문서](https://developers.google.com/identity/openid-connect/openid-connect)와 [Workspace security challenge 문서](https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges)에서는 Google 계정 내부 DB 모델과 consumer account MFA 정책 세부 구현을 확인할 수 없다.
