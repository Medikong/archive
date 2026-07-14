# ADR-002 REST, gRPC, Kafka 경계

작성일: 2026-07-14
상태: 제안

## 배경

현재 외부 사용자 명령·조회는 REST, 서비스 간 업무 상태 전파는 Kafka를 사용한다. Go 전환과 gRPC 도입은 서로 독립적인 결정이다.

## 제안

| 경계 | 기본 방식 |
| --- | --- |
| 브라우저·외부 클라이언트 → Gateway·서비스 | REST/JSON/OpenAPI |
| 주문·결제·알림 상태 변경 전파 | Kafka event |
| 새로 필요한 내부 동기 호출 | 필요성과 실패 결합도를 검증한 뒤 gRPC 검토 |

- Go로 구현해도 외부 REST를 유지한다.
- Kafka 비동기 흐름을 gRPC로 치환하지 않는다.
- gRPC는 내부 동기 호출, 강한 타입 계약, streaming 요구가 실제로 생길 때만 도입한다.

## 이유

- 브라우저는 일반 gRPC를 직접 사용하기 어려워 gRPC-Web 또는 변환 Gateway가 추가된다.
- 현재 서비스 간 핵심 흐름은 이미 Kafka로 분리돼 gRPC 전환 이점이 작다.
- 결제와 알림을 동기 gRPC로 묶으면 장애와 latency가 구매 요청에 전파될 수 있다.
- 프로토콜 변경은 DB transaction의 oversell과 outbox 문제를 해결하지 않는다.

## gRPC 도입 조건

- 호출자가 내부 서비스다.
- 즉시 응답이 필요한 동기 계약이다.
- Kafka event로 표현하면 안 되는 명확한 이유가 있다.
- timeout, retry, idempotency와 fallback 소유자가 정해졌다.
- protobuf versioning과 호환성 검증을 운영할 수 있다.

## 결과

현재 구매 구조는 `외부 REST + 내부 Kafka`를 유지한다. gRPC는 전면 전환 목표가 아니라 새 내부 동기 경계의 선택지다.
