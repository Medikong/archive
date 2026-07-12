# Naver

[네이버 로그인 API 명세](https://developers.naver.com/docs/login/api/api.md)와 [네이버 로그인 개발가이드](https://developers.naver.com/docs/login/devguide/devguide.md)를 기준으로 OAuth 2.0 로그인 API, profile 조회, token revocation과 연동 해제를 확인했다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| 네이버 로그인 API 명세 | https://developers.naver.com/docs/login/api/api.md | posts/undated-naver-login-token-revocation.md |
| 네이버 로그인 개발가이드 | https://developers.naver.com/docs/login/devguide/devguide.md | posts/undated-naver-login-token-revocation.md |

## 원문 확인 내용

- [네이버 로그인 API 명세](https://developers.naver.com/docs/login/api/api.md)는 인증 요청, 접근 토큰 발급/갱신/삭제 요청 API로 구성된다.
- [네이버 로그인 API 명세](https://developers.naver.com/docs/login/api/api.md)는 사용자가 Naver 계정 인증에 성공하면 서비스가 `code`로 access token을 발급받고, access token으로 profile API를 호출한다고 설명한다.
- [네이버 로그인 개발가이드](https://developers.naver.com/docs/login/devguide/devguide.md)는 연동 해제 API를 OAuth 2.0 RFC 7009 Token Revocation 개념으로 설명하며, access token이나 refresh token 폐기 요청 시 연결된 token 쌍까지 폐기하는 cascade 동작을 설명한다.
- [네이버 로그인 개발가이드](https://developers.naver.com/docs/login/devguide/devguide.md)는 연동 해제 후 사용자가 다시 연동 과정을 수행해야 하고, 새 사용자 동의 절차가 진행된다고 설명한다.

## 설계 포인트

- 소셜 로그인 provider 연동 해제는 단순 로그아웃이 아니라 provider 연결 관계와 token을 폐기하는 별도 작업이다.
- token revocation은 access token만 지우는 작업으로 보면 부족하다. refresh token과 provider 연결 상태까지 함께 다뤄야 한다.
- 내부 계정 탈퇴, provider 연결 해제, 현재 기기 로그아웃은 서로 다른 API로 분리하는 편이 안전하다.

## DropMong 적용 아이디어

- `DELETE /auth/social-connections/{provider}`는 현재 session logout과 분리한다.
- 계정 연결 해제 시 `external_identity.revoked_at`, `tokens.revoked_at`, `audit_event`를 함께 기록한다.
- provider 연결 해제는 민감 작업으로 보고 현재 비밀번호 또는 MFA 재확인을 요구한다.

## 확인 필요

- [Naver Login 공개 문서](https://developers.naver.com/docs/login/devguide/devguide.md)에서는 내부 계정 병합 정책과 이상 로그인 탐지 정책을 확인할 수 없다.
