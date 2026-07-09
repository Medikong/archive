# Uber

Uber 자료는 내부 service-to-service 인증과 인가 관점으로 보았다. [SPIFFE/SPIRE 글](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/)은 workload identity, [ABAC 글](https://www.uber.com/us/en/blog/attribute-based-access-control-at-uber/)은 actor/action/resource 기반 정책 모델을 설명한다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| Our Journey Adopting SPIFFE/SPIRE at Scale | https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/ | posts/undated-uber-spiffe-spire-abac.md |
| Attribute-Based Access Control at Uber | https://www.uber.com/us/en/blog/attribute-based-access-control-at-uber/ | posts/undated-uber-spiffe-spire-abac.md |

## 원문 확인 내용

- [Uber SPIFFE/SPIRE 글](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/)은 Go와 Java 공통 Auth library를 만들어 서비스 개발자가 보안 코드를 직접 많이 작성하지 않아도 되게 했다고 설명한다.
- [Uber SPIFFE/SPIRE 글](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/)은 SPIRE workload attestation에 container 기반 식별을 주로 사용하고, service name, environment, launching platform으로 workload를 구분한다고 설명한다.
- [Uber ABAC 글](https://www.uber.com/us/en/blog/attribute-based-access-control-at-uber/)은 actor, action, resource 개념을 두고, resource를 UON이라는 URI 형태로 표현한다.
- [Uber ABAC 글](https://www.uber.com/us/en/blog/attribute-based-access-control-at-uber/)은 정책을 중앙 Charter service에서 관리하고, host로 배포되어 service가 auth library를 통해 authorization decision을 평가하는 구조를 설명한다.

## 설계 포인트

- service-to-service 인증은 사용자 token과 별개로 workload identity를 가져야 한다.
- 내부 인가 정책은 사용자, 서비스, resource, action을 분리해야 확장된다.
- 대규모 조직에서는 보안 구현을 각 서비스에 흩뿌리기보다 공통 라이브러리와 정책 배포 체계로 묶는다.

## DropMong 적용 아이디어

- 초기에는 Kubernetes service account와 gateway-signed internal token으로 시작하되, 장기적으로 [SPIFFE/SPIRE](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/) 또는 service mesh mTLS를 검토한다.
- 내부 API 권한은 `actor_type=user|service`, `action`, `resource_type`, `resource_id`, `context`를 분리해 감사 로그에 남긴다.
- auth library는 token parsing, signature verification, required assurance check를 공통화한다.

## 확인 필요

- [Uber SPIFFE/SPIRE 글](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/)과 [Uber ABAC 글](https://www.uber.com/us/en/blog/attribute-based-access-control-at-uber/)에서는 실제 identity DB, policy storage schema, 서비스별 authorization rule 일부 개념만 확인된다.
