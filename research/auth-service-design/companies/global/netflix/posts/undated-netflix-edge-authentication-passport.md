# Netflix Edge Authentication and Passport

- 회사: Netflix
- 원문: https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: edge authentication, Passport, device-bound identity, MFA, 운영 가시성

## 원문 확인 내용

[Netflix Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)은 기존 로그인에서 client가 credential과 device identifier를 edge gateway인 Zuul로 보내고, API server가 backend 인증 시스템을 조율한 뒤 cookie를 발급하는 과정을 설명한다.

[Netflix Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)은 개선된 구조에서 Zuul의 identity filter가 device-bound Passport를 만들고, Passport와 Passport Action이 API와 downstream 서비스 사이에 전달된다고 설명한다. Passport는 사용자와 device identity를 담는 공통 구조로 쓰이며, 어디서 들어왔고 누가 변경했는지 추적할 수 있다.

[Netflix Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)은 edge authentication services로 token 처리와 identity resolution을 모으면서 downstream 시스템의 token 처리 부담과 지연시간이 줄었다고 설명한다. 의심스러운 연결에는 MFA를 선택적으로 도입하는 방향도 언급된다.

## 적용 아이디어

- DropMong 내부 서비스는 외부 OAuth token을 직접 해석하지 않는다.
- gateway나 auth-service가 검증한 identity context를 내부 표준 구조로 전달한다.
- identity context는 감사 로그에서 추적 가능해야 하므로 `issued_by`, `verified_at`, `source_token_type` 같은 provenance를 둔다.

## 확인 필요

- [Netflix Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)에서는 Passport Action과 Cookie Service 세부 schema가 공개되지 않았다.
