---
id: API.A.19-05
title: 주문 쿠폰 검증 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, redemption, validation]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-05 주문 쿠폰 검증

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-validations` |
| operationId | `validateCoupon` |
| 역할 | 쿠폰 하나와 주문 Snapshot을 평가해 할인·비용 Snapshot을 만든다. |
| API 유형 | Command |
| 인증·권한 | 주문 workload와 주문 사용자 binding |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | 쿠폰 하나 단위 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_05_validate_coupon.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-007~009`, `UC.A.19-23` |
| BC | `CMD.A.19-09`, `EVT.A.19-19~20`, `POLICY.A.19-06` |
| 서비스 | `ValidateCouponDiscountHandler`, [사용 Handler](../A_19_30-service/redemption-handlers.md) |

## 책임과 경계

- 주문·가격·구성 Snapshot과 사용자 쿠폰을 평가해 `CouponRedemption`의 `evaluated` 상태를 만든다.
- 주문 가격·상품·배송비 원본은 주문 Context가 소유한다.
- 여러 쿠폰의 조합 가능성을 결정하거나 여러 Aggregate를 한 요청에서 예약하지 않는다.

## 보안과 개인정보

- workload가 주문 사용자와 전달한 `user_id` binding을 증명해야 한다.
- 주문 전체 payload 대신 version·hash가 있는 SnapshotRef와 필요한 계산 항목만 받는다.
- 상품명·사용자 프로필과 결제수단을 저장·로그하지 않는다.

## 처리 규칙

1. workload, 사용자·주문 binding과 Snapshot 무결성을 확인한다.
2. 발급 당시 정책 version, 기간, 적용 대상, 최소 금액과 중지 상태를 평가한다.
3. 할인액 상한 뒤 비용 귀속 Snapshot을 계산한다.
4. `redemptionId`, 판정, 할인·비용 결과를 반환한다.

## 상태 변경과 트랜잭션

- `CouponRedemption` 평가 상태, 할인·비용 Snapshot, 원장과 outbox를 원자적으로 저장한다.
- 외부 Snapshot 조회 중 DB 트랜잭션을 오래 유지하지 않고 저장 직전 version을 다시 확인한다.
- 검증 성공은 사용 예약이나 확정을 의미하지 않는다.

## 멱등성과 동시성

- 범위는 `order_id + user_coupon_id + order_snapshot_version + Idempotency-Key`다.
- 같은 업무 키는 같은 `redemptionId`와 판정을 반환한다.
- 다른 주문 Snapshot version은 새 평가로 취급하되 활성 예약과 충돌하면 거절한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 정책·기간·대상·최소 금액 불충족 | 안정적인 reason code와 0이 아닌 명시적 거절 | 주문 화면에서 쿠폰을 해제한다. |
| Snapshot 검증 불가 | 적용 가능으로 간주하지 않음 | Snapshot을 새로 만들고 같은 업무를 재시도한다. |
| 중복 적용 정책 미결정 | 임의 조합을 허용하지 않음 | 쿠폰 하나 단위로 요청한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ValidateCouponDiscountHandler` |
| Aggregate | `CouponRedemption` |
| Repository | `CouponRedemptionRepository` |
| Port | `OrderSnapshotPort`, 사용자·상품 Snapshot Port |
| Event | `EVT.A.19-19` 또는 `EVT.A.19-20` |

## 관측성과 운영

- 정책 version, 일반화된 reason, 계산 지연과 Snapshot source version을 기록한다.
- order/user/product ID와 금액 원문은 metric label에 넣지 않는다.

## 검증 항목

- 같은 Snapshot 재요청이 같은 평가 결과를 반환한다.
- 외부 Snapshot 불일치에서 평가 상태를 성공으로 저장하지 않는다.
- 할인액과 비용 귀속 합계가 통화·상한 규칙을 지킨다.

## 연관 시퀀스

- `PAGE.A.11` 주문/결제의 쿠폰 선택 뒤 호출하며 예약 API보다 먼저 실행한다.

## 호환성과 변경 정책

- Snapshot schema와 할인 계산 변경은 version을 올리고 기존 판정을 재해석하지 않는다.

## 확인 필요

- `HOTSPOT.A.19-08`: 여러 쿠폰 조합 계약.
