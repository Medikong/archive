---
id: API.A.19-20
title: 쿠폰 이벤트 복구 재시도 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, recovery, retry]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-20 쿠폰 이벤트 복구 재시도

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-event-recoveries/{recoveryId}/retry-attempts` |
| operationId | `retryCouponEventRecovery` |
| 역할 | 승인된 복구 작업으로 새 attempt를 만들고 재실행을 접수한다. |
| API 유형 | Command |
| 인증·권한 | 운영·온콜 workload, 재처리 권한, approvalRef |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | 비동기 `202` |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_20_retry_coupon_event_recovery.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-015`, `UC.A.19-17` |
| BC | `CMD.A.19-21`, `EVT.A.19-39`, `POLICY.A.19-21~22` |
| 서비스 | `RequestCouponUseRecoveryHandler`, [운영 Worker](../A_19_30-service/operations-workers.md) |

## 책임과 경계

- `CouponEventRecovery`에 새 `attempt_id`를 만들고 `retry_pending`으로 전이한다.
- 원본 `CouponRedemption` 재실행과 결과 기록은 후속 `CMD.A.19-32`, `33`이 각각 수행한다.
- API 요청에서 원본 payload를 수정하거나 새 Domain Event를 위조하지 않는다.

## 보안과 개인정보

- approvalRef, workload 권한과 recovery의 운영 범위를 확인한다.
- 원본 payload는 저장된 `payload_ref`로만 접근하고 응답에 포함하지 않는다.

## 처리 규칙

1. recovery 상태, 현재 attempt와 approvalRef를 검증한다.
2. 새 attempt ID와 원본 업무 고유키를 연결해 retry_pending으로 바꾼다.
3. outbox Event를 저장하고 `202`로 attempt 상태 위치를 반환한다.

## 상태 변경과 트랜잭션

- `CouponEventRecovery`, attempt 원장과 outbox를 같은 트랜잭션에 저장한다.
- `CouponRedemption`은 Worker의 별도 Handler 트랜잭션에서만 변경한다.

## 멱등성과 동시성

- 범위는 `recovery_id + operation_task_ref + Idempotency-Key`다.
- 같은 key는 같은 attempt ID를 반환한다.
- 활성 attempt unique와 version 조건으로 동시 재시도 두 개를 막는다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| active attempt 존재 | 새 attempt를 만들지 않음 | 현재 attempt 상태를 조회한다. |
| 최종 실패 상태 | 재시도 가능으로 바꾸지 않음 | 새 승인 정책이 필요하다. |
| 결과 상관키 불일치 | 현재 attempt를 변경하지 않음 | 원본·결과 ref를 조사한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `RequestCouponUseRecoveryHandler` |
| Aggregate | `CouponEventRecovery` |
| Repository | `CouponEventRecoveryRepository` |
| Port | `OperationApprovalPort` |
| Event | `EVT.A.19-39` |

## 관측성과 운영

- recovery ID, attempt ID, 일반화된 결과와 재실행 lag를 기록한다.
- 업무 고유키는 로그 상관에는 사용하되 metric label로 쓰지 않는다.

## 검증 항목

- 같은 승인 작업이 attempt 하나만 만든다.
- 낡은 attempt 결과가 현재 상태를 바꾸지 않는다.
- 재실행과 결과 기록이 서로 다른 Aggregate 트랜잭션으로 분리된다.

## 연관 시퀀스

- Worker의 `CMD.A.19-32 → EVT.A.19-41 → CMD.A.19-33` 처리로 이어진다.

## 호환성과 변경 정책

- result kind 추가는 기존 소비자가 unknown을 실패로 처리하는 상태에서 version을 올린다.

## 확인 필요

- `HOTSPOT.A.19-05~06`: 재처리 권한과 횟수·간격·종료 기준.
