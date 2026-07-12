---
id: API.A.19-17
title: 쿠폰 운영 중지 적용 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, operations, stop]
source: local
created: 2026-07-11
updated: 2026-07-12
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-17 쿠폰 운영 중지 적용

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-operational-controls` |
| operationId | `applyCouponOperationalControl` |
| 역할 | 승인된 범위와 시각에 신규 발급 수량 예약·사용 예약 중지를 적용한다. |
| API 유형 | Command |
| 인증·권한 | 운영·온콜 workload, 위험 작업 권한, approvalRef |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | scope schema version 1 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_17_apply_coupon_operational_control.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-016`, `UC.A.19-15` |
| BC | `CMD.A.19-20`, `EVT.A.19-25`, `POLICY.A.19-08` |
| 서비스 | `ApplyCouponOperationalStopHandler`, [운영 Worker](../A_19_30-service/operations-workers.md) |

## 책임과 경계

- 캠페인·드롭·사용자 그룹 범위의 발급·사용 중지 상태를 기록한다.
- 적용 전 이미 확정된 수량·사용 예약은 기존 후속 Command로 닫는다.
- 승인 판단, 인시던트와 작업 감사 원본은 운영 작업 관리가 소유한다.

## 보안과 개인정보

- 위험 작업 권한과 approvalRef hash를 확인한다.
- 사용자 그룹 원본 목록이나 인시던트 문서를 받지 않고 scope ref만 저장한다.

## 처리 규칙

1. scope, 발급·사용 중지 flag, 적용 시각과 approvalRef를 검증한다.
2. 중첩 control의 우선순위와 현재 상태를 도메인 규칙으로 확인한다.
3. control과 outbox를 저장하고 gate·조회 캐시 무효화를 요청한다.

## 상태 변경과 트랜잭션

- `CouponOperationalControl`, 운영 원장과 outbox를 같은 트랜잭션에 저장한다.
- Redis 캐시 갱신 실패가 Postgres control을 되돌리지 않는다.

## 멱등성과 동시성

- 범위는 `operation_task_ref + scope + effective_at + Idempotency-Key`다.
- operation task unique와 version 조건으로 같은 중지를 중복 적용하지 않는다.
- 충돌하는 control은 도메인 우선순위 규칙 또는 명시적 충돌로 처리한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| approvalRef 무효 | 승인 원문을 공개하지 않음 | 운영 작업을 수정한다. |
| scope ref 확인 불가 | 광범위 중지로 확대하지 않음 | scope 원천 복구 뒤 재시도한다. |
| Redis 갱신 실패 | Postgres 적용 결과 유지 | Event 재처리로 캐시를 무효화한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ApplyCouponOperationalStopHandler` |
| Aggregate | `CouponOperationalControl` |
| Repository | `CouponOperationalControlRepository` |
| Port | `OperationApprovalPort`, Redis gate adapter |
| Event | `EVT.A.19-25` |

## 관측성과 운영

- operation task, scope type, 적용 결과, cache invalidation lag를 audit한다.

## 검증 항목

- 승인 없는 중지와 변조된 scope ref가 거절된다.
- 적용 이후 신규 예약이 차단되고 기존 예약은 중지 상태에 고립되지 않는다.
- Redis 장애에서도 Postgres 중지 상태가 최종 판단이다.

## 연관 시퀀스

- 운영 작업 승인 뒤 호출하며 승인 시퀀스는 외부 Context가 소유한다.

## 호환성과 변경 정책

- 새 scope type은 기존 Handler가 unknown을 거절하는 상태에서 version을 올려 추가한다.

## 결정 반영

- `HOTSPOT.A.19-04~05`: 긴급 중지는 정책 버전 변경과 분리하며 승인된 운영 작업 참조가 필요하다. 구체 승인선은 버전이 있는 위험 기반 운영 정책을 따른다.
