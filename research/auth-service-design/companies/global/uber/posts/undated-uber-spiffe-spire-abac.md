# Uber SPIFFE/SPIRE and ABAC

- 회사: Uber
- 원문: https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/
- 원문: https://www.uber.com/us/en/blog/attribute-based-access-control-at-uber/
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: workload identity, service-to-service auth, ABAC, policy distribution

## 원문 확인 내용

[Uber SPIFFE/SPIRE 글](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/)은 공통 Auth library를 Go와 Java 환경에 제공해 보안 로직을 추상화했다고 설명한다. workload 식별은 container 기반 attestation을 주로 쓰며 service name, environment, launching platform 같은 정보를 사용한다.

[Uber ABAC 글](https://www.uber.com/us/en/blog/attribute-based-access-control-at-uber/)은 actor, action, resource를 기본 모델로 두고, resource를 URI 형태의 UON으로 표현한다. 정책은 중앙 서비스에서 관리되고 host로 배포되며, 서비스는 auth library를 통해 정책 평가 API를 호출한다.

## 적용 아이디어

- DropMong 내부 서비스 인증은 사용자 access token과 분리한다.
- `service_identity`를 별도 주체로 보고, 내부 API 감사 로그에 `actor_type`과 `actor_id`를 명시한다.
- 초기 버전은 mTLS/SPIFFE를 바로 도입하지 않더라도 공통 auth middleware와 내부 claim 규격을 먼저 만든다.

## 확인 필요

- [Uber 공개 글](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/)에서는 구체적인 SPIFFE ID 형식과 Charter policy 저장 구조를 추정만 할 수 있다.
