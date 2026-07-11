---
id: API.A.19-12
title: 쿠폰 캠페인 검토 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, campaign, review]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-12 쿠폰 캠페인 검토

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-campaigns/{campaignId}/reviews` |
| operationId | `reviewCouponCampaign` |
| 역할 | 승인된 검토 작업의 승인·반려·보류 결과를 캠페인에 반영한다. |
| API 유형 | Command |
| 인증·권한 | 운영 workload, 검토 권한, approvalRef |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key`와 `expectedVersion` 필수 |
| 캐시 | `no-store` |
| 호환성 | 결과 enum 확장 정책 적용 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_12_review_coupon_campaign.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-023`, `UC.A.19-14` |
| BC | `CMD.A.19-03`, `EVT.A.19-03~05`, `POLICY.A.19-03` |
| 서비스 | `ReviewSellerCouponHandler` |

## 책임과 경계

- 외부 운영 작업의 승인 판단을 재수행하지 않고 결과와 근거 ref를 기록한다.
- 검토 결과 하나만 `CouponCampaign`에 반영한다.
- 승인 체계, 검토 문서와 감사 원본은 운영 작업 관리 시스템이 소유한다.

## 보안과 개인정보

- workload 권한과 approvalRef의 존재·hash 일치를 확인한다.
- 검토 의견 원문이나 증빙 파일을 받지 않고 reason code와 ExternalRef만 받는다.

## 처리 규칙

1. 현재 캠페인 version과 검토 가능한 상태를 확인한다.
2. approvalRef와 seller ownership Snapshot을 검증한다.
3. approved, rejected 또는 held 결과와 outbox를 기록한다.

## 상태 변경과 트랜잭션

- `CouponCampaign`, 검토 원장과 outbox를 같은 트랜잭션에 저장한다.
- 승인 원본 조회 실패를 승인으로 간주하지 않는다.

## 멱등성과 동시성

- 범위는 `campaign_id + approval_ref + expectedVersion + Idempotency-Key`다.
- 같은 승인 작업 replay는 기존 결과를 반환한다.
- 다른 결과의 병렬 검토는 version 충돌로 거절한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 승인 ref 불일치 | 원문을 공개하지 않음 | 운영 작업을 수정한다. |
| version 충돌 | 현재 version 제공 | 검토 대상을 다시 확인한다. |
| 이미 종결된 검토 | 결과를 덮어쓰지 않음 | 새 승인 작업을 만든다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ReviewSellerCouponHandler` |
| Aggregate | `CouponCampaign` |
| Repository | `CouponCampaignRepository` |
| Port | `OperationApprovalPort`, `SellerCatalogSnapshotPort` |
| Event | `EVT.A.19-03`, `04` 또는 `05` |

## 관측성과 운영

- review result, approval 검증 결과와 version 충돌을 audit로 남긴다.

## 검증 항목

- 권한 없는 workload와 변조된 approvalRef가 거절된다.
- 같은 승인 작업이 결과 Event를 중복 발행하지 않는다.
- 승인·반려 병렬 요청 중 하나만 반영된다.

## 연관 시퀀스

- 운영 작업 관리의 승인 시퀀스는 외부 Context에서 관리한다.

## 호환성과 변경 정책

- 새 review result는 기존 클라이언트의 unknown 처리 확인 뒤 추가한다.

## 확인 필요

- `HOTSPOT.A.19-05`: actor별 승인 범위와 검토 기준.
