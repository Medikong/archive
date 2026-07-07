---
id: BC.A.01
title: 주문 이벤트스토밍과 바운디드 컨텍스트
type: event-storming-bounded-context
status: example
tags: [ddd, event-storming, bounded-context, order, example]
source: local
created: 2026-07-06
updated: 2026-07-06
---

# 주문 이벤트스토밍과 바운디드 컨텍스트

## 기본 정보

- BC ID: `BC.A.01`
- 책임: 주문 생성, 주문 상태 관리, 주문 금액 스냅샷 보존.
- 사용자: 구매자, 운영자.
- 핵심 용어: 주문, 주문 라인, 주문 금액, 주문 상태.
- 제외 책임: 결제 승인, 상품 재고 원장, 쿠폰 발급.

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.01](../00-requirements/.examples/REQ_A_01_order_checkout.md) | UC 참조: [UC.A.01](../../30-uc/.examples/UC_A_01_place_order.md) | 영속성 참조: [PST.A.01](../../55-persistence/.examples/PST_A_01_order_persistence.md) | 서비스 참조: [SVC.A.01](../../60-service/.examples/SVC_A_01_order_service.md) | 시나리오 참조: [SCN.A.01](../../80-scenario/.examples/SCN_A_01_place_order.md) | 도메인 참조: [AGG.A.01](../../50-domain-model/.examples/AGG_A_01_order.md) | API 참조: [API.A.01](../../70-api/.examples/API_A_01_place_order.md)

## 컨텍스트 경계

- 이 BC가 결정하는 것: 주문 생성 가능 여부, 주문 상태, 주문 금액 스냅샷.
- 이 BC가 참조만 하는 것: 상품명, 상품 가격, 쿠폰 할인 결과, 배송지 요약.
- 다른 BC에 위임하는 것: 결제 승인, 쿠폰 유효성 최종 판정, 재고 차감.

## 이벤트스토밍

| 유형 | 식별자 | 이름 | 설명 |
| --- | --- | --- | --- |
| 사용자 행동 | UC.A.01 | 주문 확정 | 구매자가 주문 결제 페이지에서 주문을 확정한다. |
| Command | CMD.A.01 | 주문 생성 | 주문 생성 요청을 처리한다. |
| Domain Event | EVT.A.01 | 주문 생성됨 | 주문이 생성된 뒤 발행된다. |
| Policy | POLICY.A.01 | 주문 가능성 검증 | 상품 상태, 금액, 쿠폰 상태를 확인한다. |
| Read Model | RM.A.01 | 주문 결제 요약 | 주문 결제 화면에 필요한 요약 정보를 제공한다. |

## 유비쿼터스 언어

| 용어 | 의미 | 혼동하기 쉬운 용어 |
| --- | --- | --- |
| 주문 | 구매자가 상품 구매 의사를 확정한 기록 | 결제 |
| 주문 라인 | 주문에 포함된 상품 스냅샷 | 장바구니 항목 |
| 주문 금액 | 주문 생성 시점에 확정한 서버 계산 금액 | 화면 표시 금액 |

## 소속 도메인 모델

- Aggregate: [AGG.A.01](../../50-domain-model/.examples/AGG_A_01_order.md)
- Entity: `ENT.A.01`
- Value Object: `VO.A.01`
- Read Model: `RM.A.01`

## 제공 API

- [API.A.01](../../70-api/.examples/API_A_01_place_order.md)

## 구현 서비스

- [SVC.A.01](../../60-service/.examples/SVC_A_01_order_service.md)

## 영속성 설계

- [PST.A.01](../../55-persistence/.examples/PST_A_01_order_persistence.md)

## 발행 Event

- `EVT.A.01`
- `EVT.A.02`

## 열린 질문

- 쿠폰 할인 결과를 주문 생성 전 조회 모델에 포함할지, 주문 생성 시 동기 검증으로만 처리할지 결정해야 한다.
