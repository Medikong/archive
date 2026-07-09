# Naver Login Token Revocation

- 회사: Naver
- 원문: https://developers.naver.com/docs/login/api/api.md
- 원문: https://developers.naver.com/docs/login/devguide/devguide.md
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: OAuth 2.0, access token, refresh token, token revocation, 연동 해제

## 원문 확인 내용

[네이버 로그인 API 명세](https://developers.naver.com/docs/login/api/api.md)는 인증 요청, 접근 토큰 발급/갱신/삭제 요청으로 구성된다. 서비스는 사용자가 인증에 성공한 뒤 전달받은 `code`로 access token을 발급받고, 그 token으로 profile API를 호출한다.

[네이버 로그인 개발가이드](https://developers.naver.com/docs/login/devguide/devguide.md)는 연동 해제를 Token Revocation으로 설명한다. `/oauth2.0/revoke`에 access token 또는 refresh token을 지정하면 해당 token뿐 아니라 연결된 token 쌍도 함께 폐기된다. 연동 해제 후에는 사용자가 다시 동의 과정을 거쳐야 한다.

## 적용 아이디어

- DropMong도 `logout`, `logout all devices`, `revoke refresh token`, `unlink social provider`를 분리한다.
- provider 연결 해제는 내부 `external_identity` 상태 변경과 외부 provider revocation 호출을 함께 다루는 saga 후보가 된다.
- cascade revocation은 `refresh_token_family_id` 단위 무효화 모델에 반영한다.

## 확인 필요

- [Naver Login 공개 문서](https://developers.naver.com/docs/login/devguide/devguide.md)에서는 provider subject 변경, email 변경, 중복 계정 병합 운영 방식을 확인할 수 없다.
