# Kakao Login OAuth/OIDC

- 회사: Kakao
- 원문: https://developers.kakao.com/docs/en/kakaologin/common
- 원문: https://developers.kakao.com/docs/en/kakaologin/rest-api
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: OAuth 2.0, OpenID Connect, token, service user id, consent

## 원문 확인 내용

[Kakao Login REST API](https://developers.kakao.com/docs/en/kakaologin/rest-api)는 서비스 서버가 authorization code를 받고 token endpoint에서 access token과 refresh token을 받는 구조를 설명한다. [Kakao Login Concepts](https://developers.kakao.com/docs/en/kakaologin/common)는 OIDC를 활성화하면 ID token도 함께 발급된다고 설명한다.

[Kakao Login REST API](https://developers.kakao.com/docs/en/kakaologin/rest-api)에서 로그인 후 처리 책임은 서비스 서버에 남아 있다. 서비스 서버는 access token으로 Kakao 사용자 정보를 조회하고, 조회한 service user id가 내부 서비스 회원인지 판단한 뒤 로그인 또는 가입 처리를 한다.

[Kakao Login Concepts](https://developers.kakao.com/docs/en/kakaologin/common)는 REST API access token 만료가 6시간, refresh token 만료가 2개월이라고 설명한다. 같은 문서는 refresh token을 갱신하면 새 refresh token이 발급되고 이전 refresh token은 폐기된다고 설명한다.

## 적용 아이디어

- 외부 provider id는 내부 회원 id가 아니라 `external_identity`의 `provider_subject`로 저장한다.
- 내부 로그인 완료는 `auth-service`가 외부 인증을 확인한 뒤 `member-service`의 회원 상태를 확인하는 과정으로 나눈다.
- refresh token rotation을 기본으로 두고, 이전 refresh token 재사용을 보안 이벤트로 기록한다.

## 확인 필요

- [Kakao Login 공개 문서](https://developers.kakao.com/docs/en/kakaologin/common)에서는 Kakao의 내부 감사 로그, 이상 로그인 탐지, token family 모델을 확인할 수 없다.
