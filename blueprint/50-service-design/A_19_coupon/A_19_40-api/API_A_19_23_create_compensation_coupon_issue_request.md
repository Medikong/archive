---
id: API.A.19-23
title: 보상 쿠폰 발급 요청 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, compensation, issuance, cs]
source: local
created: 2026-07-11
updated: 2026-07-12
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-23 보상 쿠폰 발급 요청

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/compensation-coupon-issue-requests` |
| operationId | `createCompensationCouponIssueRequest` |
| 역할 | 증빙과 위험 기반 승인 조건을 충족한 CS·운영 보상 작업을 공통 쿠폰 발급 요청으로 변환한다. |
| API 유형 | Command |
| 인증·권한 | CS·운영 workload, compensation 권한, `caseRef`; 위험 정책이 요구할 때 `approvalRef` |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | 비동기 `202` |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_23_create_compensation_coupon_issue_request.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-021`, `UC.A.19-20` |
| BC | `CMD.A.19-13`, `EVT.A.19-07`, `POLICY.A.19-01` |
| 서비스 | `CreateCouponIssueRequestHandler`, [발급 Handler](../A_19_30-service/issuance-handlers.md) |

## 책임과 경계

- `source_type=compensation`, `caseRef`, 사유, 주문 또는 인시던트 ref와 승인 정책 스냅샷을 가진 `CouponIssueRequest`를 만든다.
- 실제 `UserCoupon` 생성, 수량 확정과 발급 완료 기록은 후속 Policy가 처리한다.
- 보상 한도·승인 판단과 CS 문의 원문은 외부 시스템이 소유한다. Context 쿠폰은 버전이 있는 위험 정책과 승인 참조의 일치만 확인한다.

## 보안과 개인정보

- compensation 권한과 `caseRef` binding을 항상 확인하고, 고액·대량 또는 정책 한도 초과 요청에는 `approvalRef`를 요구한다.
- 문의 내용·증빙·사용자 프로필을 요청·Event·로그에 넣지 않는다.
- target user ID는 workload 범위 안에서만 허용한다.

## 처리 규칙

1. workload, target user, campaign, `caseRef`, 사유와 주문 또는 인시던트 참조를 검증하고 위험 정책이 요구하면 `approvalRef`를 확인한다.
2. 동일 보상 작업 ref의 기존 발급 요청을 확인한다.
3. `CouponIssueRequest`, 원장과 outbox를 저장하고 `202`로 상태 위치를 반환한다.

## 상태 변경과 트랜잭션

- `CouponIssueRequest`, 보상 source 원장과 outbox를 원자적으로 저장한다.
- `UserCoupon`과 `CouponCampaign` 수량은 후속 Command가 각각 변경한다.

## 멱등성과 동시성

- 범위는 `approval_ref + case_ref + target_user_id + campaign_id + Idempotency-Key`다.
- 보상 작업 ref unique로 다른 key의 중복 요청도 차단한다.
- 같은 key는 기존 issueRequest를 반환한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 승인·case binding 불일치 | 원문을 공개하지 않음 | 외부 작업을 수정한다. |
| 캠페인 발급 불가 | 임의 대체 쿠폰을 발급하지 않음 | 승인 작업에서 대상을 다시 정한다. |
| 비동기 발급 실패 | 성공으로 위조하지 않음 | 발급 복구 정책과 상태 조회를 사용한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `CreateCouponIssueRequestHandler` |
| Aggregate | `CouponIssueRequest` |
| Repository | `CouponIssueRequestRepository` |
| Port | `OperationApprovalPort`, 사용자 자격 Snapshot Port |
| Event | `EVT.A.19-07` |

## 관측성과 운영

- approval/case ref hash, issueRequest, 결과와 처리 지연을 audit한다.

## 검증 항목

- 같은 보상 작업이 발급 요청 하나만 만든다.
- 승인·문의 원문과 개인정보가 쿠폰 저장소에 복제되지 않는다.
- 실제 발급 실패를 보상 요청 성공과 구분한다.

## 연관 시퀀스

- 외부 CS 승인 뒤 호출하고 발급 처리 시퀀스는 일반 발급과 같다.

## 호환성과 변경 정책

- 보상 source subtype 추가는 승인 계약과 함께 version을 올린다.

## 결정 반영

- `HOTSPOT.A.19-05`: `caseRef`, 사유와 주문 또는 인시던트 참조가 필수다. 고액·대량 또는 정책 한도 초과 보상은 운영 승인을 요구하며 구체 한도는 버전이 있는 운영 정책으로 관리한다.
