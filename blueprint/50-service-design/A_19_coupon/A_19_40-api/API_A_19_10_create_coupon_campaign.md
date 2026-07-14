---
id: API.A.19-10
title: 쿠폰 캠페인 등록 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, campaign, policy]
source: local
created: 2026-07-11
updated: 2026-07-12
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-10 쿠폰 캠페인 등록

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/coupon-campaigns` |
| operationId | `createCouponCampaign` |
| 역할 | 혜택·적용 조건·기간·발급 및 비용 주체를 가진 캠페인을 등록한다. |
| API 유형 | Command |
| 인증·권한 | 판매자·마케팅·운영 workload와 소유·권한 스냅샷. 위험 정책이 요구할 때 `approvalRef` |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | policy schema version 1 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_10_create_coupon_campaign.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-001`, `FR-021~022`, `UC.A.19-07`, `UC.A.19-09` |
| BC | `CMD.A.19-01`, `EVT.A.19-01~02`, `POLICY.A.19-01~02` |
| 서비스 | `RegisterCouponPolicyHandler`, [발급 Handler](../A_19_30-service/issuance-handlers.md) |

## 책임과 경계

- `CouponCampaign`에 `CouponBenefit`, `CouponApplicabilityPolicy`, 책임 주체, 승인 정책 스냅샷과 선택적 템플릿 참조를 등록한다.
- 상품·드롭·판매자 원본을 소유하지 않고 검증 SnapshotRef만 보존한다.
- 선착순 수량은 `API.A.19-11`, 승인 결과는 `API.A.19-12`가 별도 Command로 처리한다.

## 보안과 개인정보

- workload의 actor type과 issuer·cost bearer 권한을 대조하고, 위험 정책이 요구하는 요청에는 `approvalRef`를 확인한다.
- 판매자 소유 Snapshot의 version·hash 불일치 시 등록하지 않는다.
- 상품 전체 payload와 승인 문서를 저장하지 않는다.

## 처리 규칙

1. actor, 발급·비용 주체, 기간, 혜택과 승인 정책 스냅샷의 완결성을 검사하고 위험 정책에 따라 승인 필요 여부를 판단한다.
2. 적용 대상의 외부 소유 Snapshot을 확인한다.
3. 정책 version 1을 가진 캠페인과 필요 시 검토 요청 Event를 만든다.

## 상태 변경과 트랜잭션

- 캠페인, 정책 Snapshot, 원장과 outbox를 같은 트랜잭션에 저장한다.
- 외부 소유 조회 중 트랜잭션을 열어 두지 않고 저장 전 Snapshot version을 재확인한다.

## 멱등성과 동시성

- 범위는 `API.A.19-10 + actor/workload + external_business_ref + Idempotency-Key`다.
- 동일 key replay는 같은 campaign을 반환하고 다른 정책 payload는 충돌로 거절한다.
- 외부 업무 ref unique가 다른 key의 중복 캠페인을 방지한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 소유·책임 주체 불일치 | 내부 소유 구조를 숨김 | 외부 Snapshot과 권한을 갱신한다. |
| 정책 입력 불완전 | 위반 필드만 공개 | 정책을 보완한다. |
| Snapshot 원천 장애 | 임의 소유로 등록하지 않음 | 같은 key로 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `RegisterCouponPolicyHandler` |
| Aggregate | `CouponCampaign` |
| Entity·VO | `CouponBenefit`, `CouponApplicabilityPolicy` |
| Repository | `CouponCampaignRepository` |
| Port | `SellerCatalogSnapshotPort` |
| Event | `EVT.A.19-01`, 필요 시 `EVT.A.19-02` |

## 관측성과 운영

- actor type, 정책 schema version, 결과와 Snapshot 검증 지연을 기록한다.

## 검증 항목

- 발급·비용 주체 누락과 판매자 소유 불일치가 등록되지 않는다.
- 같은 외부 업무 ref 동시 요청이 캠페인 하나만 만든다.
- outbox 실패가 캠페인 저장과 분리되지 않는다.

## 연관 시퀀스

- 판매자·운영 화면 시퀀스는 아직 없다.

## Context 판매자 연동 상태

- 이 Endpoint의 seller 사용 범위와 현재 구현 차이는 [Coupon 판매자 연동 계약 gap](seller-contract-gaps.md)을 따른다. signed seller scope는 현재 membership·permission 재검증을 대신하지 않는다.

## 호환성과 변경 정책

- 혜택·적용 정책 의미 변경은 policy schema version을 올린다.

## 결정 반영

- `HOTSPOT.A.19-05`: 판매자 전액 부담·자기 소유 범위·승인된 템플릿은 판매자 권한으로 요청할 수 있다. 플랫폼·공동 부담, 제휴, 템플릿 초과와 고액·대량 보상은 운영 승인을 요구한다.
