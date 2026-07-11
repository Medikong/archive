---
id: API.A.19-14
title: 대량 쿠폰 발급 작업 등록 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, bulk, issuance]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-14 대량 쿠폰 발급 작업 등록

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/bulk-coupon-issue-jobs` |
| operationId | `createBulkCouponIssueJob` |
| 역할 | 불변 대상 정의 ref와 기준 시각을 가진 대량 발급 작업을 접수한다. |
| API 유형 | Command |
| 인증·권한 | 마케팅·운영 workload, approvalRef |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | 비동기 `202` |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_14_create_bulk_coupon_issue_job.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-019`, `UC.A.19-11` |
| BC | `CMD.A.19-08`, `EVT.A.19-16~18` |
| 서비스 | `RegisterBulkCouponIssueJobHandler`, [운영 Worker](../A_19_30-service/operations-workers.md) |

## 책임과 경계

- 대상 정의 ref, 평가 기준 시각과 캠페인을 가진 `BulkCouponIssueJob`을 등록한다.
- 요청에서 사용자 목록 원문을 받거나 여러 사용자 쿠폰을 직접 만들지 않는다.
- 대상 페이지 조회와 개별 발급은 Worker가 공통 `CMD.A.19-13`을 요청한다.

## 보안과 개인정보

- approvalRef와 대상 정의 ref 소유 workload를 확인한다.
- 대상 사용자 목록과 세그먼트 조건 원문을 쿠폰 DB·로그에 복제하지 않는다.

## 처리 규칙

1. 캠페인 상태, approvalRef, 대상 정의 ref와 기준 시각을 검증한다.
2. 작업을 registered 상태로 저장한다.
3. `Location`, 작업 ID와 권장 조회 간격을 `202 Accepted`로 반환한다.

## 상태 변경과 트랜잭션

- `BulkCouponIssueJob`, 작업 원장과 outbox를 같은 트랜잭션에 저장한다.
- 개별 `CouponIssueRequest`는 Worker의 별도 트랜잭션에서 생성한다.

## 멱등성과 동시성

- 범위는 `campaign_id + audience_ref + evaluation_as_of + Idempotency-Key`다.
- 외부 업무 ref unique로 같은 작업을 중복 등록하지 않는다.
- Worker는 page cursor와 대상 business key로 재개한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 대상·승인 ref 무효 | 원본 조건을 공개하지 않음 | 외부 작업을 수정한다. |
| 캠페인 발급 불가 | 작업을 만들지 않음 | 캠페인 상태를 확인한다. |
| Worker 지연 | 접수 실패로 바꾸지 않음 | 작업 상태와 lag를 조회한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `RegisterBulkCouponIssueJobHandler` |
| Aggregate | `BulkCouponIssueJob` |
| Repository | `BulkCouponIssueJobRepository` |
| Port | `OperationApprovalPort`, `BulkAudiencePort` |
| Event | `EVT.A.19-16` |

## 관측성과 운영

- 작업 상태, 대상 page 처리량, 성공·거절·최종 실패와 lag를 기록한다.

## 검증 항목

- 같은 업무 ref가 작업 하나만 만든다.
- 대상 사용자 원문이 쿠폰 저장소와 로그에 복제되지 않는다.
- Worker 재시작 뒤 cursor와 기준 시각으로 이어서 처리한다.

## 연관 시퀀스

- API 접수 뒤 Worker 실행은 [운영 Worker](../A_19_30-service/operations-workers.md)를 따른다.

## 호환성과 변경 정책

- 대상 정의 schema는 외부 원천이 version을 소유하고 쿠폰은 ref만 받는다.

## 확인 필요

- `HOTSPOT.A.19-06`: 대상 평가 시점과 재처리·종료 기준.
