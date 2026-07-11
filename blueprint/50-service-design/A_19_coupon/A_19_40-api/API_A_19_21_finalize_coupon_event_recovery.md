---
id: API.A.19-21
title: 쿠폰 이벤트 복구 최종 실패 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, recovery, finalization]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-21 쿠폰 이벤트 복구 최종 실패

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-event-recoveries/{recoveryId}/finalization` |
| operationId | `finalizeCouponEventRecovery` |
| 역할 | 승인된 운영 작업으로 사용 이벤트 복구를 `failed_final`로 종료한다. |
| API 유형 | Command |
| 인증·권한 | 운영 workload, 최종 실패 권한, approvalRef |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1` |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_21_finalize_coupon_event_recovery.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-015`, `UC.A.19-17` |
| BC | `CMD.A.19-25`, `EVT.A.19-40` |
| 서비스 | `FinalizeCouponUseRecoveryHandler` |

## 책임과 경계

- 사용 이벤트 복구 Aggregate를 최종 실패로 닫고 승인·사유 ref를 기록한다.
- 원본 `CouponRedemption` 상태를 되돌리거나 성공으로 정정하지 않는다.
- 발급 실패의 최종 종료 `CMD.A.19-22`는 발급 Worker 내부 정책이며 이 Endpoint 대상이 아니다.

## 보안과 개인정보

- 최종 실패 권한과 approvalRef 무결성을 확인한다.
- 실패 payload·문의·인시던트 원문 대신 reason code와 ref만 저장한다.

## 처리 규칙

1. recovery 상태와 활성 attempt 부재를 확인한다.
2. approvalRef와 final reason을 검증한다.
3. failed_final 상태, 원장과 outbox를 저장한다.

## 상태 변경과 트랜잭션

- `CouponEventRecovery`, 최종 실패 원장과 outbox를 원자적으로 저장한다.
- 원본 redemption과 다른 Aggregate를 같은 트랜잭션에 넣지 않는다.

## 멱등성과 동시성

- 범위는 `recovery_id + operation_task_ref + Idempotency-Key`다.
- 같은 승인 작업 replay는 기존 failed_final 결과를 반환한다.
- 활성 attempt 생성과 finalization 경쟁은 version·상태 조건으로 하나만 성공한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| active attempt 존재 | 종료하지 않음 | attempt 완료·취소 뒤 재판단한다. |
| 이미 복구 성공 | final failure로 덮어쓰지 않음 | 현재 result_ref를 확인한다. |
| 승인 ref 무효 | 원문을 공개하지 않음 | 운영 작업을 수정한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `FinalizeCouponUseRecoveryHandler` |
| Aggregate | `CouponEventRecovery` |
| Repository | `CouponEventRecoveryRepository` |
| Port | `OperationApprovalPort` |
| Event | `EVT.A.19-40` |

## 관측성과 운영

- final reason, approval 검증, 시도 수와 처리 기간을 audit한다.

## 검증 항목

- 복구 성공 상태가 최종 실패로 바뀌지 않는다.
- active attempt와 finalization이 동시에 성공하지 않는다.
- 승인 원문과 payload가 로그·Event에 복제되지 않는다.

## 연관 시퀀스

- `API.A.19-19` 조회와 운영 승인 뒤 호출한다.

## 호환성과 변경 정책

- final reason code의 기존 의미를 변경하지 않고 새 code만 추가한다.

## 확인 필요

- `HOTSPOT.A.19-05~06`: 최종 실패 승인자와 종료 기준.
