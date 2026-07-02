# DropMong 아키텍처 설계 패킷

이 폴더는 기존 Ticketmong/Medikong 문서를 직접 덮어쓰기 전에, DropMong 전환 설계를 합의하기 위한 별도 패킷이다. 문서의 목적은 구현자가 서비스 코드, GitOps, 인프라 작업을 시작하기 전에 필요한 결정을 빠짐없이 확인하게 만드는 것이다.

## DropMong이란

DropMong은 한정 수량 상품을 정해진 시간에 공개하고, 짧은 시간에 몰리는 주문을 안정적으로 처리하는 드롭 커머스 서비스다. 고객은 오픈 예정 드롭을 둘러보다가 오픈 시각에 주문과 결제를 진행하고, 운영자는 상품, 드롭 일정, 판매 수량, 장애 상태를 관리한다.

이 서비스의 핵심은 예쁜 상품 목록보다 **피크 순간의 정합성**이다. 많은 사용자가 동시에 주문해도 판매 가능 수량보다 많은 주문이 확정되면 안 되고, 결제 실패나 알림 장애가 발생해도 재고와 주문 상태는 복구 가능해야 한다.

기존 Ticketmong이 공연 좌석 예매 흐름에 가까웠다면, DropMong은 좌석, 티켓 발행, 공연 회차 개념을 덜어내고 상품 드롭, 재고 예약, 주문 확정, 결제 이벤트, 알림으로 도메인을 다시 잡는다.

## 설계의 핵심 문장

```text
DropMong은 제한 수량 드롭 커머스이며, 핵심 불변 조건은 oversell 0이다.
재고 예약과 주문 상태 변경은 order-service가 단독 소유한다.
외부 진입은 Istio Ingress Gateway를 사용하며 Kong은 제거한다.
서비스 수는 auth, catalog, order, payment, notification 5개로 시작한다.
```

## 구현 전 필수 문서

| 순서 | 문서 | 목적 |
| --- | --- | --- |
| 00 | [00-product-scope.md](./00-product-scope.md) | 어떤 서비스인지, 만들 것과 만들지 않을 것 고정 |
| 01 | [01-domain-model.md](./01-domain-model.md) | 도메인 용어, 상태, 불변 조건 정의 |
| 02 | [02-system-architecture.md](./02-system-architecture.md) | 전체 시스템 구조와 repo 영향 정의 |
| 03 | [03-service-boundaries.md](./03-service-boundaries.md) | 5개 서비스 책임 경계 고정 |
| 04 | [04-data-design.md](./04-data-design.md) | DB, transaction, idempotency, outbox 설계 |
| 05 | [05-api-contracts.md](./05-api-contracts.md) | HTTP API와 에러 계약 |
| 06 | [06-event-contracts.md](./06-event-contracts.md) | Kafka topic, event envelope, DLQ 규칙 |
| 07 | [07-critical-flows.md](./07-critical-flows.md) | 주문, 결제, 만료, 장애, rollback 흐름 |
| 08 | [08-infra-deployment.md](./08-infra-deployment.md) | Istio, GitOps, HPA, KEDA, Rollouts |
| 09 | [09-observability-slo.md](./09-observability-slo.md) | SLO, metric, log, trace, alert |
| 10 | [10-security.md](./10-security.md) | 인증, 인가, mTLS, secret, audit |
| 11 | [11-test-release-plan.md](./11-test-release-plan.md) | 테스트, canary, rollback 완료 기준 |
| 12 | [12-user-flows.md](./12-user-flows.md) | 고객, 운영자, 플랫폼 운영자 사용자 흐름 |

## 보조 자료

- [01-visual-architecture.md](./01-visual-architecture.md): 초기 합의용 압축 설계 요약.
- [architecture-visual.html](./architecture-visual.html): 브라우저에서 바로 여는 데스크톱 기준 설계 보드.
- [assets/service-map.svg](./assets/service-map.svg): Markdown과 발표 자료에 재사용할 수 있는 서비스 맵 이미지.
- [adr/](./adr/): 흔들리기 쉬운 결정 기록.
- [runbooks/](./runbooks/): 장애 대응 초안.
- [diagrams/](./diagrams/): Mermaid 다이어그램 원본.
- [DESIGN.md](./DESIGN.md): HTML 보드의 색상, 타이포, 컴포넌트 기준.

## 권장 작성 순서

1. `00-product-scope.md`부터 `03-service-boundaries.md`까지 읽고 제품과 서비스 경계를 합의한다.
2. `04-data-design.md`, `05-api-contracts.md`, `06-event-contracts.md`로 구현 계약을 확정한다.
3. `07-critical-flows.md`, `08-infra-deployment.md`, `09-observability-slo.md`로 운영 경로를 맞춘다.
4. `12-user-flows.md`로 화면과 사용자 행동 흐름을 맞춘다.
5. `10-security.md`, `11-test-release-plan.md`로 릴리즈 게이트를 잠근다.
6. ADR을 accepted 상태로 바꾼 뒤 `services`, `e-gitops`, `infra` 변경을 시작한다.

## 브라우저로 보기

`architecture-visual.html`은 외부 CDN 없이 동작하는 단일 HTML 파일이다. 데스크톱 브라우저에서 문서 순서와 핵심 구조를 확인하는 용도다.
