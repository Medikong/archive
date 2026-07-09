# Google OIDC, OAuth, IAP

- 회사: Google
- 원문: https://developers.google.com/identity/openid-connect/openid-connect
- 원문: https://developers.google.com/identity/protocols/oauth2
- 원문: https://docs.cloud.google.com/iap/docs/concepts-overview
- 원문: https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: ID token 검증, refresh token, 중앙 authorization 계층, security challenge

## 원문 확인 내용

[Google OIDC 문서](https://developers.google.com/identity/openid-connect/openid-connect)는 ID token의 서명, issuer, audience, expiry를 확인해야 한다고 설명한다. [Google OAuth 2.0 문서](https://developers.google.com/identity/protocols/oauth2)는 authorization code를 access token과 refresh token으로 교환하고, refresh token은 안전한 장기 저장소에 보관해야 한다고 설명한다.

[Google Cloud IAP 문서](https://docs.cloud.google.com/iap/docs/concepts-overview)는 application 앞에 중앙 authorization 계층을 둔다. [Workspace security challenge 문서](https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges)는 의심 로그인이나 민감 작업에서 추가 확인을 요구하는 방식을 설명한다.

## 적용 아이디어

- 외부 ID token 검증과 내부 session 발급을 같은 transaction으로 묶되, 책임은 분리한다.
- API gateway는 인증 결과를 내부 서비스에 전달하고, 내부 서비스는 사용자 인증 원문 token을 직접 파싱하지 않게 한다.
- 민감 작업 challenge는 action 단위로 `verified_until`을 짧게 둔다.

## 확인 필요

- [IAP 문서](https://docs.cloud.google.com/iap/docs/concepts-overview)는 Google Cloud 제품 문서다. DropMong이 직접 같은 제품을 쓴다는 의미는 아니고, 중앙 인증/인가 계층 패턴으로 참고한다.
