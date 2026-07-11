---
id: API.A.19-15
title: 대량 쿠폰 발급 작업 조회 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, bulk, query]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-15 대량 쿠폰 발급 작업 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/internal/bulk-coupon-issue-jobs/{bulkJobId}` |
| operationId | `getBulkCouponIssueJob` |
| 역할 | 작업 진행 상태와 대상별 집계 결과를 조회한다. |
| API 유형 | Query |
| 인증·권한 | 등록 workload 또는 허가된 운영 workload |
| 노출 범위 | internal |
| 멱등성 | 해당 없음 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1` |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_15_get_bulk_coupon_issue_job.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-019`, `UC.A.19-11` |
| Read Model | `RM.A.19-04`, `RM.A.19-05` 요약 |
| 서비스 | [운영 Worker](../A_19_30-service/operations-workers.md) |

## 책임과 경계

- registered, running, completed, failed 상태와 집계 수를 반환한다.
- 개별 사용자 목록, 자격 Snapshot과 실패 payload 원문을 반환하지 않는다.
- 상세 실패 조사는 `API.A.19-19` 복구 목록을 사용한다.

## 보안과 개인정보

- 작업 소유 workload 또는 운영 권한을 확인한다.
- 사용자 ID와 대상 조건 원문은 집계 응답에서 제외한다.

## 처리 규칙

1. 작업 접근 권한과 ID를 검증한다.
2. 작업 Aggregate projection과 집계 Read Model을 조회한다.
3. `asOf`, 진행 cursor 요약과 결과 집계를 반환한다.

## 상태 변경과 트랜잭션

- 상태를 변경하지 않는다.
- 비동기 projection 지연을 `asOf`로 드러낸다.

## 멱등성과 동시성

- Query이므로 멱등 키가 필요 없다.
- 집계 Event는 event ID unique로 한 번만 반영한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 작업 없음·권한 없음 | 동일한 공개 범위 | 등록 결과 Location을 확인한다. |
| 투영 지연 | 완료로 추측하지 않음 | Retry-After 뒤 재조회한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Service | 대량 작업 Query |
| Aggregate·Read Model | `BulkCouponIssueJob`, 성과·실패 집계 |
| Repository | 작업·Read Model Query Repository |

## 관측성과 운영

- 상태별 조회량, projection lag, 작업 처리율을 기록한다.

## 검증 항목

- 집계 합계가 성공·거절·최종 실패 원장과 일치한다.
- 권한 없는 workload가 작업 존재를 추측하지 못한다.
- 재처리 Event가 집계를 중복 증가시키지 않는다.

## 연관 시퀀스

- `API.A.19-14`의 `Location`이 이 Endpoint를 가리킨다.

## 호환성과 변경 정책

- 새 집계 필드는 선택 항목으로 추가한다.

## 확인 필요

- `HOTSPOT.A.19-06`: 최종 실패 판정과 목표 완료 기준.
