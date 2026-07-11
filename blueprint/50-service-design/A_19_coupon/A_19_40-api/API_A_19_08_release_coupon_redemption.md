---
id: API.A.19-08
title: 쿠폰 사용 예약 해제 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, redemption, release]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-08 쿠폰 사용 예약 해제

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-redemptions/{redemptionId}/release` |
| operationId | `releaseCouponRedemption` |
| 역할 | 확정 전 예약을 원인 결과 ref와 함께 해제한다. |
| API 유형 | Command |
| 인증·권한 | 주문 workload와 예약 order binding |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1` |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_08_release_coupon_redemption.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-011`, `UC.A.19-25` |
| BC | `CMD.A.19-12`, `EVT.A.19-23` |
| 서비스 | `ReleaseCouponRedemptionHandler`, [사용 Handler](../A_19_30-service/redemption-handlers.md) |

## 책임과 경계

- `reserved → released` 전이만 수행한다.
- 확정된 사용은 해제하지 않고 `API.A.19-09` 회수 계약을 사용한다.
- 재사용 허용 여부와 유예 시간은 서버 정책에 맡기고 wire 상수로 고정하지 않는다.

## 보안과 개인정보

- 예약의 주문 binding과 해제 원인 ref를 검증한다.
- 외부 실패 payload·고객 문의 원문 대신 reason code와 ExternalRef만 받는다.

## 처리 규칙

1. 예약과 주문 binding, 해제 원인 SnapshotRef를 확인한다.
2. 현재 상태가 reserved이면 released로 바꾼다.
3. 쿠폰함 투영과 운영 결과용 outbox를 기록한다.

## 상태 변경과 트랜잭션

- `CouponRedemption`, 사용 원장과 outbox를 같은 트랜잭션에 저장한다.
- 사용자 쿠폰 상태는 직접 수정하지 않는다.

## 멱등성과 동시성

- 범위는 `redemption_id + release_ref + Idempotency-Key`다.
- 같은 ref의 released 상태는 기존 결과를 반환한다.
- commit과 경쟁하면 Aggregate version·상태 조건으로 하나만 성공한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 이미 confirmed | release로 위장하지 않음 | 회수 가능 여부를 별도 판단한다. |
| 원인 ref 검증 실패 | 예약을 유지 | 원천 결과를 확인하고 재시도한다. |
| 결과 불명 | 복구 기록으로 분리 | 같은 업무 고유키로 재실행한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ReleaseCouponRedemptionHandler` |
| Aggregate | `CouponRedemption` |
| Repository | `CouponRedemptionRepository` |
| Event | `EVT.A.19-23` |

## 관측성과 운영

- reason code, 전이 결과, version 충돌과 처리 지연을 기록한다.

## 검증 항목

- 확정 사용을 release하지 않는다.
- commit/release 동시 요청에서 종단 상태가 하나다.
- 같은 ref 재요청이 원장을 중복 추가하지 않는다.

## 연관 시퀀스

- 주문 생성·결제 실패·주문 만료 시나리오와 연결되며 구체 유예는 아직 없다.

## 호환성과 변경 정책

- 새 release reason 추가는 하위 호환으로 처리하고 기존 의미를 재사용하지 않는다.

## 확인 필요

- `HOTSPOT.A.19-03`: 해제 유예와 예약 중 만료 처리 시점.
