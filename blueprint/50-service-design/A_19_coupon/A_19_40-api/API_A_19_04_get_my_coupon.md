---
id: API.A.19-04
title: 내 쿠폰 상세 조회 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, detail, query]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-04 내 쿠폰 상세 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/users/me/coupons/{userCouponId}` |
| operationId | `getMyCoupon` |
| 역할 | 본인 쿠폰의 혜택, 상태, 기간과 적용 조건을 조회한다. |
| API 유형 | Query |
| 인증·권한 | 사용자 Principal과 쿠폰 소유권 |
| 노출 범위 | public |
| 멱등성 | 해당 없음 |
| 캐시 | `private, no-store` |
| 호환성 | `/api/v1` |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_04_get_my_coupon.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-007~008`, `UC.A.19-03` |
| Read Model | `RM.A.19-02` |
| 도메인 | [캠페인과 정책](../A_19_10-domain-model/campaign-policy.md), [발급](../A_19_10-domain-model/issuance.md) |

## 책임과 경계

- 발급 당시 정책 Snapshot을 기준으로 혜택·적용 범위·기간과 현재 표시 상태를 제공한다.
- 상품·드롭 원본을 소유하거나 현재 판매 상태를 확정하지 않는다.
- 실제 주문 적용 가능 여부와 할인 금액은 `API.A.19-05`가 주문 Snapshot으로 다시 평가한다.

## 보안과 개인정보

- 현재 Principal 소유가 아니면 존재 여부를 구분하지 않는 `404`로 반환한다.
- 외부 대상은 허용된 표시 Snapshot만 반환하고 내부 ref·payload hash는 숨긴다.

## 처리 규칙

1. `userCouponId`와 Principal 소유 범위를 확인한다.
2. 쿠폰 상세 Read Model과 최신 활성 안내를 조회한다.
3. `asOf`와 함께 표시 가능한 조건·상태를 반환한다.

## 상태 변경과 트랜잭션

- 상태를 변경하지 않는다.
- 조회 시 정책을 재계산해 발급 Snapshot을 덮어쓰지 않는다.

## 멱등성과 동시성

- Query이므로 멱등 키가 필요 없다.
- Read Model version과 `asOf`로 투영 시점을 드러낸다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 없음·소유권 불일치 | 같은 `404` 사용 | 쿠폰함을 다시 조회한다. |
| 외부 표시 Snapshot 누락 | 적용 가능으로 추측하지 않음 | 서비스 복구 뒤 재조회한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Service | 쿠폰 상세 Query |
| Read Model | `CouponDetailAndApplicability` |
| Repository | 쿠폰 상세 Query Repository |
| 원천 | `CouponCampaign`, `UserCoupon` Event 투영 |

## 관측성과 운영

- 조회 결과, projection version과 lag를 기록하고 소유 식별자는 label에서 제외한다.

## 검증 항목

- 다른 사용자 ID로 상세를 추측할 수 없다.
- 발급 정책 Snapshot과 현재 표시 상태가 섞이지 않는다.
- 적용 불가 상세가 내부 상품·판매자 정보를 과도하게 노출하지 않는다.

## 연관 시퀀스

- 쿠폰함 상세 화면과 주문 전 표시에서 사용하며 실제 주문 검증은 `API.A.19-05`다.

## 호환성과 변경 정책

- 표시 필드는 선택적으로 추가한다. 할인 계산 의미 변경은 policy schema version이 필요하다.

## 확인 필요

- `HOTSPOT.A.19-04`: 정책 변경이 기존 쿠폰 표시와 사용에 미치는 영향.
