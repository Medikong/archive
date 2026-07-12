---
id: SD.A.1910
title: Context 쿠폰 도메인 모델 설계 인덱스
type: service-design-domain-model-index
status: draft
tags: [service-design, coupon, domain-model, index]
source: local
created: 2026-07-09
updated: 2026-07-12
service_design: SD.A.19
bounded_context: BC.A.19
---

# Context 쿠폰 도메인 모델 설계 인덱스

## 역할

Context 쿠폰의 8개 Aggregate를 책임별 문서로 안내한다. Aggregate의 필드, 상태 전이, 불변조건과 계약 정의는 하위 문서에서만 관리한다.

## 원천

- [BC.A.19 Context 쿠폰](../../../40-event-storming-bounded-context/BC_A_19_coupon.md)
- [REQ.A.02 쿠폰 및 혜택](../../../00-requirements/REQ_A_02_coupon_benefit.md)
- [SD.A.19 서비스 상세 설계](../README.md)
- 보조 근거: [기존 도메인 모델](../../../../../workspaces/docs/architecture/coupon-service/01-domain-model.md), [기존 엔티티 설계](../../../../../workspaces/docs/architecture/coupon-service/02-entity-design.md)

## 하위 문서

| 문서 | 책임 | 주요 Aggregate | 상태 |
| --- | --- | --- | --- |
| [캠페인과 정책](campaign-policy.md) | 혜택, 적용 조건, 발급 제한, 승인과 정책 버전 | `CouponCampaign` | draft |
| [발급](issuance.md) | 코드, 발급 요청, 사용자 쿠폰 생성 | `CouponCodeBatch`, `CouponIssueRequest`, `UserCoupon` | draft |
| [사용](redemption.md) | 주문 적용 검증, 사용 예약·확정·해제·회수, 비용 귀속 | `CouponRedemption` | draft |
| [운영과 복구](operations-recovery.md) | 대량 발급, 운영 제어, 만료, 실패 이벤트 복구 | `BulkCouponIssueJob`, `CouponOperationalControl`, `CouponEventRecovery` | draft |
| [공통 계약](shared-contracts.md) | 공통 ID/VO, Event·Policy·Rule·Read Model 계약, 외부 경계 | 전체 | draft |

## 결정 경계

- Context 쿠폰은 사용자·상품·드롭·주문·결제·CS·정산 원본을 소유하지 않는다. 외부 식별자와 판단 시점의 스냅샷만 보존한다.
- 한 Command는 한 Aggregate만 변경한다. 다른 Aggregate의 후속 변경은 Event와 Policy로 연결한다.
- 발급 수량은 `issue_request_id`별 예약에서 확정 또는 해제로 한 번만 전이한다.
- 사용 복구는 `recovery_id`, `attempt_id`, 업무 고유키, `result_ref`를 함께 보존한다.
- `HOTSPOT.A.19-01~08`과 `09`의 개인정보 원칙은 [결정 기록](../hotspot-decisions.md)을 따른다. 자동 지급 생산자·Event 유형·채널처럼 남은 계약은 결정 기록에 명시된 범위만 열린 상태로 둔다.
