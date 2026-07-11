---
id: API.A.19-06
title: 쿠폰 사용 예약 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, redemption, reserve]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-06 쿠폰 사용 예약

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-redemptions/{redemptionId}/reserve` |
| operationId | `reserveCouponRedemption` |
| 역할 | 검증된 쿠폰 평가를 같은 주문의 활성 사용 예약으로 바꾼다. |
| API 유형 | Command |
| 인증·권한 | 주문 workload와 원본 주문 binding |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1` |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_06_reserve_coupon_redemption.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-009~010`, `UC.A.19-06` |
| BC | `CMD.A.19-10`, `EVT.A.19-21` |
| 서비스 | `ReserveCouponRedemptionHandler`, [사용 Handler](../A_19_30-service/redemption-handlers.md) |

## 책임과 경계

- `evaluated` 또는 허용된 `released` 상태 하나를 `reserved`로 전이한다.
- 주문 생성·재고 예약·결제 승인을 수행하지 않는다.
- 사용자 쿠폰의 화면 상태는 Event를 반영한 Read Model이 제공한다.

## 보안과 개인정보

- 검증 때 저장한 order/user binding과 호출 workload를 다시 대조한다.
- 다른 주문의 redemption 존재와 상태를 공개하지 않는다.

## 처리 규칙

1. 현재 평가, 주문 Snapshot version과 운영 중지를 확인한다.
2. 동일 사용자 쿠폰의 다른 활성 예약이 없는지 확인한다.
3. 예약 상태와 outbox를 저장하고 예약 결과를 반환한다.

## 상태 변경과 트랜잭션

- `CouponRedemption`, 사용 원장과 outbox를 같은 트랜잭션에 저장한다.
- `UserCoupon`을 직접 변경하지 않는다.
- 중지 적용 전에 이미 확정된 수량·평가는 현재 설계의 운영 중지 경계를 따른다.

## 멱등성과 동시성

- 범위는 `redemption_id + order_id + Idempotency-Key`다.
- 활성 예약 partial unique와 row lock으로 다른 주문의 동시 예약을 차단한다.
- 같은 key는 기존 reserved 결과를 반환한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 평가 만료·version 불일치 | 상태 충돌로 반환 | 다시 검증한다. |
| 다른 활성 예약 존재 | 내부 주문 ID를 숨김 | 기존 주문을 정리하거나 다른 쿠폰을 선택한다. |
| 운영 중지 | 신규 예약을 만들지 않음 | 안내와 운영 상태를 확인한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ReserveCouponRedemptionHandler` |
| Aggregate | `CouponRedemption` |
| Repository | `CouponRedemptionRepository` |
| Event | `EVT.A.19-21` |

## 관측성과 운영

- 예약 결과, 충돌 유형, Aggregate version과 지연을 관측한다.

## 검증 항목

- 같은 사용자 쿠폰이 두 주문에 동시에 예약되지 않는다.
- 중지·평가 만료에서 상태가 변경되지 않는다.
- 응답 유실 뒤 같은 key가 같은 예약을 반환한다.

## 연관 시퀀스

- 검증 뒤, 주문 확정 전에 호출한다.

## 호환성과 변경 정책

- 예약 TTL을 wire 상수로 고정하지 않고 서버가 `reservedUntil`을 반환한다.

## 확인 필요

- `HOTSPOT.A.19-03`: 예약 해제 유예와 예약 중 만료 처리 시점.
