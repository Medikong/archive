# LINE Login OIDC ID Token

- 회사: LINE
- 원문: https://developers.line.biz/en/docs/line-login/integrate-line-login/
- 원문: https://developers.line.biz/en/docs/line-login/verify-id-token/
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: OAuth 2.0 authorization code, OIDC, ID token, signature validation

## 원문 확인 내용

[LINE Login v2.1 문서](https://developers.line.biz/en/docs/line-login/integrate-line-login/)는 OAuth 2.0 authorization code grant와 OIDC를 지원한다고 설명한다. web app은 callback URL을 등록하고, 사용자 인증과 인가 뒤 authorization code와 `state`를 받는다.

[LINE ID token 문서](https://developers.line.biz/en/docs/line-login/verify-id-token/)는 ID token이 JWT이며 서비스가 보안상 검증해야 한다고 설명한다. 같은 문서는 서명 검증, issuer, subject, audience, expiry, nonce 같은 claim을 확인하는 구조를 설명한다.

## 적용 아이디어

- DropMong은 ID token을 세션 대체물로 장기 보관하지 않는다. 최초 로그인 검증에 사용하고 내부 session/token을 별도로 발급한다.
- `oauth_login_attempt`를 만들고 `state`와 `nonce`를 일회성으로 소비한다.
- provider별 ID token claim은 `external_identity.last_claims_hash`처럼 최소 정보만 남기고 원문 token은 저장하지 않는다.

## 확인 필요

- [LINE Login 문서](https://developers.line.biz/en/docs/line-login/integrate-line-login/)는 provider 연동 관점이다. DropMong 내부 회원 상태, 제재, 탈퇴, 재가입 정책은 별도로 설계해야 한다.
