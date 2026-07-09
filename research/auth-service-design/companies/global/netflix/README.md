# Netflix

Netflix TechBlog의 [Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)을 기준으로 edge authentication, token-agnostic identity propagation, Passport 모델을 확인했다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| Edge Authentication and Token-Agnostic Identity Propagation | https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602 | posts/undated-netflix-edge-authentication-passport.md |

## 원문 확인 내용

- [Netflix Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)은 사용자 credential과 device identifier가 edge gateway인 Zuul로 들어오고, 인증 후 cookie가 발급되는 과정을 설명한다.
- [Netflix Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)은 이후 구조에서 Zuul의 identity filter가 device-bound Passport를 만들고, API와 mid-tier 서비스가 Passport를 전달받아 사용자와 기기 identity를 해석한다고 설명한다.
- [Netflix Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)은 Passport를 canonical identity 구조로 설명하며, provenance를 추적할 수 있어 identity 관련 이상 징후를 디버깅하기 쉬워진다고 설명한다.
- [Netflix Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)은 token 처리를 edge authentication services로 모아 downstream 서비스의 CPU, latency, GC 부담을 줄였다고 설명한다.
- [Netflix Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)은 말미에서 의심스러운 연결에 MFA를 선택적으로 도입하는 방향을 언급한다.

## 설계 포인트

- 내부 서비스가 외부 token 종류를 각각 해석하면 복잡도와 보안 편차가 커진다.
- edge/gateway에서 token을 검증하고, 내부에는 정규화된 identity envelope을 전달하는 패턴이 대규모 microservice에 적합하다.
- identity 구조에는 사용자뿐 아니라 device, trust level, provenance가 들어가야 운영 분석이 가능하다.

## DropMong 적용 아이디어

- 내부 서비스는 provider token을 직접 받지 않고 `internal_identity_context`만 신뢰한다.
- identity context에는 `member_id`, `session_id`, `device_id`, `auth_time`, `assurance_level`, `issuer`, `provenance`를 둔다.
- DropMong 초기에는 [Netflix식 대규모 Passport 서비스](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)까지 만들기보다 gateway-signed claim과 공통 검증 라이브러리로 시작한다.

## 확인 필요

- [Netflix Edge Authentication 글](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)에서는 Passport의 구체적 DB schema와 cookie service 내부 구현을 확인할 수 없다.
