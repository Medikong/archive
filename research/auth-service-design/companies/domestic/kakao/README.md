# Kakao

[Kakao Login Concepts](https://developers.kakao.com/docs/en/kakaologin/common)와 [Kakao Login REST API](https://developers.kakao.com/docs/en/kakaologin/rest-api)를 기준으로 OAuth 2.0, OIDC, 토큰 만료, 서비스 회원 판단 경계를 확인했다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| Kakao Login Concepts | https://developers.kakao.com/docs/en/kakaologin/common | posts/undated-kakao-login-oauth-oidc.md |
| Kakao Login REST API | https://developers.kakao.com/docs/en/kakaologin/rest-api | posts/undated-kakao-login-oauth-oidc.md |

## 원문 확인 내용

- [Kakao Login REST API](https://developers.kakao.com/docs/en/kakaologin/rest-api)는 authorization code를 받은 뒤 token endpoint에서 access token과 refresh token을 발급하는 OAuth 2.0 구조를 따른다.
- [Kakao Login Concepts](https://developers.kakao.com/docs/en/kakaologin/common)는 OpenID Connect를 활성화하면 ID token이 추가로 발급된다고 설명한다.
- [Kakao Login Concepts](https://developers.kakao.com/docs/en/kakaologin/common)는 REST API 기준 access token 만료를 6시간, refresh token 만료를 2개월로 설명하고, 갱신 시 새 refresh token이 발급되며 이전 refresh token은 폐기된다고 설명한다.
- [Kakao Login REST API](https://developers.kakao.com/docs/en/kakaologin/rest-api)는 서비스 서버가 Kakao access token으로 사용자 정보를 조회한 뒤, 해당 Kakao service user id가 내부 서비스 회원인지 판단한다고 설명한다.
- [Kakao Login Concepts](https://developers.kakao.com/docs/en/kakaologin/common)는 REST API key에 client secret 기능이 기본 활성화될 수 있고, token 발급 요청에서 client secret을 요구할 수 있다고 설명한다.

## 설계 포인트

- 외부 provider의 `service user id`를 내부 `member_id`로 직접 쓰지 않고, `external_identity(provider, provider_user_id)`로 보관하는 편이 안전하다.
- Kakao refresh token 교체 정책은 DropMong의 refresh token rotation 모델을 설계할 때 참고할 수 있다.
- OIDC ID token을 쓰더라도 `nonce`와 서명 검증, issuer/audience/expiry 검증은 서비스 서버 책임으로 남겨야 한다.

## DropMong 적용 아이디어

- `auth_credentials`와 `external_identities`를 분리하고, Kakao/Naver/LINE 같은 provider별 subject를 같은 테이블로 수용한다.
- 로그인 완료 후 내부 회원 가입 여부, 약관 동의, 휴면/정지 상태는 `member-service`가 판단한다.
- refresh token 갱신 시 이전 token을 폐기하고 `rotated_from_token_id`, `revoked_at`, `reuse_detected_at`을 기록한다.

## 확인 필요

- [Kakao Login 공개 문서](https://developers.kakao.com/docs/en/kakaologin/common)만으로는 Kakao 내부 계정 DB 구조, 감사 로그, rate limit 운영 방식을 확인하지 못했다.
