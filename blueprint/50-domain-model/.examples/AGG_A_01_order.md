---
id: AGG.A.01
title: 주문 Aggregate와 Entity
type: domain-model
status: example
tags: [ddd, aggregate, order, example]
source: local
created: 2026-07-06
updated: 2026-07-06
---

# 주문 Aggregate와 Entity

## 기본 정보

- Aggregate ID: `AGG.A.01`
- Aggregate Root: `Order`
- 소속 BC: [BC.A.01](../../40-event-storming-bounded-context/.examples/BC_A_01_order.md)
- 책임: 주문 생성, 주문 금액 스냅샷 보존, 주문 상태 전이.
- 생명주기: 생성됨 -> 결제 대기 -> 결제 완료 또는 취소.

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.01](../00-requirements/.examples/REQ_A_01_order_checkout.md) | UC 참조: [UC.A.01](../../30-uc/.examples/UC_A_01_place_order.md) | 영속성 참조: [PST.A.01](../../55-persistence/.examples/PST_A_01_order_persistence.md) | 서비스 참조: [SVC.A.01](../../60-service/.examples/SVC_A_01_order_service.md) | 시나리오 참조: [SCN.A.01](../../80-scenario/.examples/SCN_A_01_place_order.md) | API 참조: [API.A.01](../../70-api/.examples/API_A_01_place_order.md) | BC 참조: [BC.A.01](../../40-event-storming-bounded-context/.examples/BC_A_01_order.md)

## 모델 개요

```mermaid
classDiagram
    class Order {
        +orderId
        +buyerId
        +status
        +totalAmount
        +place()
    }

    class OrderLine {
        +orderLineId
        +productId
        +quantity
        +unitPrice
    }

    class Money {
        +amount
        +currency
    }

    Order "1" --> "*" OrderLine
    Order --> Money
```

## Aggregate Root

| 필드 | 타입 | 설명 | 상태 |
| --- | --- | --- | --- |
| orderId | UUID | 주문 식별자 | 확인 |
| buyerId | UUID | 구매자 식별자 | 확인 |
| status | OrderStatus | 주문 상태 | 확인 |
| totalAmount | Money | 서버가 계산한 최종 금액 | 확인 |

## Entity

| Entity ID | 이름 | 책임 | 식별자 | Aggregate 내 관계 |
| --- | --- | --- | --- | --- |
| `ENT.A.01` | OrderLine | 주문 상품 스냅샷 보존 | orderLineId | Order 하위 |

## 불변조건

- 주문은 하나 이상의 주문 라인을 가져야 한다.
- 주문 생성 시 최종 금액은 서버 계산 결과를 따른다.
- 생성된 주문 라인의 상품명과 가격은 이후 상품 변경과 무관하게 보존한다.

## Event

- `EVT.A.01`: 주문 생성 완료.
- `EVT.A.02`: 주문 취소 완료.

## Repository

| Repository | 메서드 | 책임 | 영속성 근거 |
| --- | --- | --- | --- |
| `OrderRepository` | `save(order)` | 주문 Aggregate 저장 | [PST.A.01](../../55-persistence/.examples/PST_A_01_order_persistence.md) |
| `OrderRepository` | `findById(orderId)` | 주문 Aggregate 복원 | [PST.A.01](../../55-persistence/.examples/PST_A_01_order_persistence.md) |
