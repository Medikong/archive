---
id: API.A.19-09
title: 쿠폰 사용 회수 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, redemption, revoke]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-09 쿠폰 사용 회수

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-redemptions/{redemptionId}/revoke` |
| operationId | `revokeCouponRedemption` |
| 역할 | 확정된 쿠폰 사용을 취소·환불 결과에 따라 회수하고 비용을 보정한다. |
| API 유형 | Command |
| 인증·권한 | 주문·결제 또는 승인된 운영 workload |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | 재사용 허용 의미는 미확정 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_09_revoke_coupon_redemption.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-011`, `FR-017`, `UC.A.19-19`, `UC.A.19-25` |
| BC | `CMD.A.19-15`, `EVT.A.19-24`, `EVT.A.19-28` |
| 서비스 | `ReclaimCouponRedemptionHandler`, [사용 Handler](../A_19_30-service/redemption-handlers.md) |

## 책임과 경계

- `confirmed → reclaimed` 전이와 기존 비용 귀속의 보정 사실을 기록한다.
- 환불·취소 승인과 고객 문의 원본은 외부 Context가 소유한다.
- 회수 뒤 쿠폰을 다시 사용할 수 있는지는 이 API가 결정하지 않는다.

## 보안과 개인정보

- 주문 결과 또는 승인된 operation ref와 workload 권한을 검증한다.
- CS 문의·환불 payload 원문을 받지 않고 ExternalRef와 reason code만 받는다.

## 처리 규칙

1. confirmed 상태와 원본 확정 결과를 확인한다.
2. 취소·환불 또는 승인된 운영 결과 ref를 검증한다.
3. reclaimed 전이, 보정 원장과 outbox를 저장한다.

## 상태 변경과 트랜잭션

- `CouponRedemption`, 사용·비용 보정 원장과 outbox를 원자적으로 저장한다.
- 정산 상태를 직접 변경하지 않고 비용 보정 Event를 발행한다.

## 멱등성과 동시성

- 범위는 `redemption_id + revoke_ref + Idempotency-Key`다.
- 같은 ref의 재요청은 기존 reclaimed 결과를 반환한다.
- 서로 다른 취소 ref는 원본 주문 결과 version과 상태 조건으로 충돌 처리한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| confirmed 아님 | 현재 상태를 과도하게 공개하지 않음 | 예약이면 release 계약을 사용한다. |
| 승인·결과 ref 불일치 | 회수를 수행하지 않음 | 외부 작업 원본을 수정한 뒤 재요청한다. |
| 정산 전달 실패 | 회수 상태를 되돌리지 않음 | 같은 outbox Event를 재전달한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ReclaimCouponRedemptionHandler` |
| Aggregate | `CouponRedemption` |
| Repository | `CouponRedemptionRepository` |
| Port | `OperationApprovalPort`, `SettlementEventPort` |
| Event | `EVT.A.19-24`, `EVT.A.19-28` |

## 관측성과 운영

- 회수 reason, 승인 ref 검증 결과, 비용 보정 Event 지연을 기록한다.

## 검증 항목

- 같은 취소 결과가 회수·비용 보정을 한 번만 만든다.
- 미확정 예약을 회수로 처리하지 않는다.
- 승인 원본 장애를 성공으로 위조하지 않는다.

## 연관 시퀀스

- 주문 취소·환불 또는 승인된 CS 작업 뒤 실행한다.

## 호환성과 변경 정책

- 회수 뒤 재사용 여부를 추가할 때는 응답 status와 Read Model 계약을 새 version으로 검토한다.

## 확인 필요

- `HOTSPOT.A.19-03`: 회수 유예와 회수 뒤 재사용 허용 기준.
