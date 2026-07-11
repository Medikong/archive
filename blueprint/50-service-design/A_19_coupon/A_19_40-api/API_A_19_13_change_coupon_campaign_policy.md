---
id: API.A.19-13
title: 쿠폰 정책 변경 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, campaign, policy-version]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-13 쿠폰 정책 변경

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-campaigns/{campaignId}/policy-versions` |
| operationId | `createCouponCampaignPolicyVersion` |
| 역할 | 미래 적용 시각을 가진 새 정책 version을 등록한다. |
| API 유형 | Command |
| 인증·권한 | 운영 workload와 approvalRef |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key`와 `expectedVersion` 필수 |
| 캐시 | `no-store` |
| 호환성 | 과거 version 불변 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_13_change_coupon_campaign_policy.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-020`, `UC.A.19-13` |
| BC | `CMD.A.19-04`, `EVT.A.19-06`, `POLICY.A.19-07` |
| 서비스 | `ChangeCouponPolicyHandler` |

## 책임과 경계

- 기존 정책을 수정하지 않고 새 version과 미래 `effectiveAt`을 추가한다.
- 기존 사용자 쿠폰과 활성 예약에 새 version을 적용할지 추측하지 않는다.
- 승인 판단과 영향 분석 원본은 운영 시스템이 소유한다.

## 보안과 개인정보

- approvalRef와 정책 변경 권한, expected version을 확인한다.
- 영향 분석 문서 원문 대신 ref·hash만 보존한다.

## 처리 규칙

1. 현재 version, 새 정책 schema와 미래 적용 시각을 검증한다.
2. 승인 원본 ref의 무결성을 확인한다.
3. 새 version과 outbox를 기록하고 투영·캐시 무효화를 요청한다.

## 상태 변경과 트랜잭션

- `CouponCampaign`, version 원장과 outbox를 원자적으로 저장한다.
- 과거 정책 Snapshot과 기존 UserCoupon Snapshot을 덮어쓰지 않는다.

## 멱등성과 동시성

- 범위는 `campaign_id + new_policy_version + Idempotency-Key`다.
- expected version 조건부 갱신과 `(campaign_id, policy_version)` unique를 사용한다.
- 같은 version의 다른 payload는 충돌이다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 과거·현재 적용 시각 | 업무 거절 | 미래 시각으로 수정한다. |
| version 충돌 | 현재 version 제공 | 새 version 번호로 재작성한다. |
| 승인 ref 장애 | 변경을 저장하지 않음 | 승인 원천 복구 뒤 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ChangeCouponPolicyHandler` |
| Aggregate | `CouponCampaign` |
| Repository | `CouponCampaignRepository` |
| Port | `OperationApprovalPort` |
| Event | `EVT.A.19-06` |

## 관측성과 운영

- old/new version, effectiveAt, approval 검증 결과와 투영 지연을 기록한다.

## 검증 항목

- 같은 version에 다른 정책이 저장되지 않는다.
- 기존 쿠폰 Snapshot이 소급 변경되지 않는다.
- 캐시 무효화 실패 뒤 Postgres version으로 복구된다.

## 연관 시퀀스

- 정책 적용 전 영향 분석·승인은 외부 운영 시퀀스가 담당한다.

## 호환성과 변경 정책

- 기존 version은 불변이고 새 의미는 새 version으로만 추가한다.

## 확인 필요

- `HOTSPOT.A.19-04`: 기존 사용자 쿠폰·진행 중 예약에 적용할 version.
