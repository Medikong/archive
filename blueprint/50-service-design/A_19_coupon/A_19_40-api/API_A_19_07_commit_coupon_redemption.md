---
id: API.A.19-07
title: 쿠폰 사용 확정 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, redemption, commit]
source: local
created: 2026-07-11
updated: 2026-07-12
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-07 쿠폰 사용 확정

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-redemptions/{redemptionId}/commit` |
| operationId | `commitCouponRedemption` |
| 역할 | 같은 주문의 활성 예약을 확정 사용으로 기록한다. |
| API 유형 | Command |
| 인증·권한 | 주문·결제 workload와 결과 참조 binding |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | 확정 원천 의미는 Hotspot 경계 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_07_commit_coupon_redemption.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-010`, `UC.A.19-24` |
| BC | `CMD.A.19-11`, `EVT.A.19-22`, `EVT.A.19-28` |
| 서비스 | `ConfirmCouponRedemptionHandler`, [사용 Handler](../A_19_30-service/redemption-handlers.md) |

## 책임과 경계

- `reserved → confirmed` 전이와 비용 귀속 사실을 기록한다.
- 주문·결제 상태를 소유하거나 PG 결과를 재판정하지 않는다.
- 호출자는 합의된 확정 결과 ref와 Snapshot version을 제공한다.

## 보안과 개인정보

- 원본 예약의 order binding과 결과 ref 소유 workload를 확인한다.
- 결제수단, PG payload와 주문 원문을 받거나 저장하지 않는다.

## 처리 규칙

1. 예약 상태, order binding과 결과 SnapshotRef를 확인한다.
2. `ConfirmCouponRedemptionHandler`가 확정과 비용 귀속 Snapshot을 기록한다.
3. 정산·알림·쿠폰함용 outbox Event를 저장한다.

## 상태 변경과 트랜잭션

- `CouponRedemption`, 사용·비용 원장과 outbox를 원자적으로 저장한다.
- 정산 상태를 변경하지 않고 비용 귀속 Event만 발행한다.

## 멱등성과 동시성

- 범위는 `redemption_id + order_result_ref + Idempotency-Key`다.
- 이미 confirmed이고 result ref가 같으면 기존 결과를 반환한다.
- 다른 ref의 확정·해제·회수 경쟁은 Aggregate version으로 거절한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 예약 없음·해제됨 | 현재 상태 충돌만 공개 | 주문 상태를 확인한다. |
| 결과 ref 검증 불가 | 확정으로 간주하지 않음 | 원천 조회 복구 뒤 같은 key로 재시도한다. |
| 결과 불명 장애 | 별도 복구 기록 생성 | `recovery_id`로 운영 복구를 추적한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ConfirmCouponRedemptionHandler` |
| Aggregate | `CouponRedemption` |
| Repository | `CouponRedemptionRepository` |
| Port | `OrderSnapshotPort`, `SettlementEventPort` |
| Event | `EVT.A.19-22`, `EVT.A.19-28` |

## 관측성과 운영

- 확정 결과, result ref 검증 결과, 비용 Event 지연과 version 충돌을 관측한다.

## 검증 항목

- 같은 결과 재요청이 비용 원장을 중복 기록하지 않는다.
- 해제와 확정 동시 요청 중 하나의 종단 전이만 성공한다.
- 정산 포트 실패가 확정 트랜잭션을 되돌리지 않고 outbox로 재전달된다.

## 연관 시퀀스

- 결제 최종 확정 사건을 원천으로 연결한다. 주문 생성이나 중간 결제 승인만으로는 이 API를 호출하지 않는다.

## 호환성과 변경 정책

- 확정 원천 event type 변경은 새 Snapshot schema version과 함께 배포한다.

## 결정 반영

- `HOTSPOT.A.19-02`: 결제 최종 확정 사건을 검증한 뒤에만 예약을 사용 완료로 바꾼다.
