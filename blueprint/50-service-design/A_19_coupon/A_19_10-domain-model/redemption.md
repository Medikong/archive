---
id: SD.A.1910.03
title: Context 쿠폰 사용 도메인 모델
type: service-design-domain-model
status: draft
tags: [service-design, coupon, redemption, settlement]
source: local
created: 2026-07-10
updated: 2026-07-12
service_design: SD.A.19
bounded_context: BC.A.19
domain_model: SD.A.1910
---

# Context 쿠폰 사용 도메인 모델

## 책임

`CouponRedemption`이 주문 스냅샷에 대한 적용 가능성 판단, 사용 예약·확정·해제·회수와 비용 귀속을 한 사용 원장으로 관리하는 방법을 정의한다. 주문·결제와 정산 원본 상태는 소유하지 않는다.

## 연관 문서

- 원천: [BC.A.19](../../../40-event-storming-bounded-context/BC_A_19_coupon.md), [REQ.A.02](../../../00-requirements/REQ_A_02_coupon_benefit.md)
- 결정: [Context 쿠폰 Hotspot 결정 기록](../hotspot-decisions.md)
- 도메인: [캠페인과 정책](campaign-policy.md), [발급](issuance.md), [공통 계약](shared-contracts.md)
- 구현 설계: [쓰기 모델](../A_19_20-persistence/write-models.md), [사용 Handler](../A_19_30-service/redemption-handlers.md)

## Aggregate 구성

| 모델 | 종류 | 책임 |
| --- | --- | --- |
| `CouponRedemption` | Aggregate Root (`AGG.A.19-04`) | 한 사용자 쿠폰과 주문의 검증·예약·확정·해제·회수 상태를 보호한다. |
| `OrderSnapshot` | Value Object | 주문·결제 Context가 제공한 상품, 드롭, 판매자, 금액, 사용자 자격과 기준 시각을 보존한다. |
| `DiscountSnapshot` | Value Object | 정책 버전, 할인 전후 금액, 통화, 계산 항목과 적용·거절 사유를 보존한다. |
| `CostAttribution` | Value Object | 플랫폼·판매자·공동 부담·보상 비용과 승인·정산 참조를 보존한다. |
| `RedemptionResultRef` | Value Object | 멱등 재실행이 반환할 기존 결과 Event 또는 원장 식별자를 묶는다. |

## 상태 전이

```mermaid
stateDiagram-v2
  [*] --> Evaluated
  Evaluated --> Rejected: 적용 불가
  Evaluated --> Reserved: 사용 예약
  Reserved --> Confirmed: 사용 확정
  Reserved --> Released: 예약 해제
  Confirmed --> Reclaimed: 사용 회수
```

- 적용 가능 확인은 할인 계산 결과를 만들지만 쿠폰을 점유하지 않는다.
- 예약은 사용자 쿠폰 하나를 주문 하나에 점유한다.
- 확정 전 실패·취소·만료는 `release`, 확정 뒤 취소·환불은 `reclaim`으로 구분한다.
- 확정 실패·취소가 검증되면 예약을 즉시 해제한다. 결과가 불명확한 경우에만 버전이 있는 운영 설정의 짧은 유예를 적용한다.
- 사용 완료 쿠폰을 다시 사용할 수 있게 하려면 검증된 취소·환불 사건, 남은 유효기간, 캠페인 활성 상태와 운영 중지 부재를 모두 확인해야 한다. 기존 `CouponRedemption`을 역전하지 않고 새 예약 Command가 새 사용 원장을 만든다.
- 사용 확정은 결제 최종 확정 사건의 원본 참조를 검증한 뒤에만 수행한다. 주문 생성이나 중간 결제 승인만으로 확정하지 않는다.

## 주문 스냅샷 검증

검증 입력은 `order_id`, `user_id`, 상품별 가격·수량, 배송비, 판매자·상품·드롭·카테고리 외부 참조, 사용자 자격, 평가 시각을 포함한다. Context 쿠폰은 다음만 수행한다.

1. `UserCoupon`의 소유 사용자, 사용 기간과 자체 상태를 확인한다.
2. 저장된 `policy_version`과 외부 참조를 주문 스냅샷에 대조한다.
3. 기본적으로 할인 쿠폰 한 장과 배송비 쿠폰 한 장만 함께 허용한다. 그 밖의 조합은 버전이 있는 `stackingPolicyRef`가 명시적으로 허용하는지 검증한다.
4. 할인 금액과 비용 귀속 스냅샷을 서버에서 계산한다.
5. 상품·드롭의 존재나 주문 금액 원본을 수정하지 않는다.

## 불변조건

- 하나의 `UserCoupon`에는 활성 `reserved` 사용 원장이 최대 하나다.
- `(order_id, user_coupon_id, operation_type, business_key)`의 같은 작업은 한 번만 반영한다.
- `confirmed`가 아닌 사용은 회수할 수 없고, `reserved`가 아닌 사용은 해제할 수 없다.
- 할인 금액은 0 이상이며 주문 할인 대상 금액을 넘지 않는다.
- 비용 귀속 합계는 할인 금액과 같아야 한다.
- 사용 Command는 `CouponRedemption`만 변경한다. 쿠폰함 표시와 정산 전달은 Event 투영으로 처리한다.
- 할인 적용 순서는 상품·판매자 범위 할인, 주문 할인, 배송비 할인이다. 명시되지 않은 플랫폼·판매자·드롭·회원 쿠폰 조합은 거절한다.

## BC 추적

| 유형 | ID | 이 문서의 책임 |
| --- | --- | --- |
| Aggregate | `AGG.A.19-04` | `CouponRedemption` |
| Command | `CMD.A.19-09`, `CMD.A.19-10`, `CMD.A.19-11`, `CMD.A.19-12`, `CMD.A.19-15` | 검증, 예약, 확정, 해제, 회수 |
| Event | `EVT.A.19-19`, `EVT.A.19-20`, `EVT.A.19-21`, `EVT.A.19-22`, `EVT.A.19-23`, `EVT.A.19-24`, `EVT.A.19-28` | 적용·사용 결과와 비용 귀속 |
| Policy | `POLICY.A.19-06`, `POLICY.A.19-08` | 주문 스냅샷 검증과 운영 중지 우선 |
| Business Rule | `RULE.A.19-03`, `RULE.A.19-04`, `RULE.A.19-07`, `RULE.A.19-10`, `RULE.A.19-11` | 활성 주문, 멱등성, 비용, 표시 합성, 전이 구분 |

## 결정 반영

| Hotspot | 영향 |
| --- | --- |
| `HOTSPOT.A.19-02` | 예약에서 잠그고 결제 최종 확정 사건 뒤에만 사용 완료로 바꾼다. |
| `HOTSPOT.A.19-03` | 확정 실패·취소는 즉시 해제하고 불명확한 결과에만 짧은 유예를 적용한다. 재사용은 검증 조건을 모두 만족해야 한다. |
| `HOTSPOT.A.19-08` | 할인 한 장과 배송비 한 장을 기본으로 허용하고 그 밖의 조합은 `stackingPolicyRef`로만 허용한다. |
