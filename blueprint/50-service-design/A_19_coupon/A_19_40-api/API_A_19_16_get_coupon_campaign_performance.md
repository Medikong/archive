---
id: API.A.19-16
title: 쿠폰 캠페인 성과 조회 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, campaign, performance, query]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-16 쿠폰 캠페인 성과 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/internal/coupon-campaigns/{campaignId}/performance` |
| operationId | `getCouponCampaignPerformance` |
| 역할 | 요청·발급·실패·사용·회수 집계를 운영 범위에서 조회한다. |
| API 유형 | Query |
| 인증·권한 | 마케팅·운영 workload |
| 노출 범위 | internal |
| 멱등성 | 해당 없음 |
| 캐시 | `no-store` |
| 호환성 | seller 전용 범위 제외 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_16_get_coupon_campaign_performance.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-013~014`, `UC.A.19-12` |
| Read Model | `RM.A.19-04` |
| 영속성 | [조회 모델과 인덱스](../A_19_20-persistence/read-models-and-indexes.md) |

## 책임과 경계

- 캠페인별 발급 요청, 실제 발급, 거절·실패, 사용·해제·회수 수를 반환한다.
- 사용자별 원장이나 정산 확정 상태를 반환하지 않는다.
- 판매자 전용 집계 범위는 현재 Endpoint에 포함하지 않는다.

## 보안과 개인정보

- workload role과 캠페인 접근 범위를 확인한다.
- user/order ID를 집계나 metric label로 노출하지 않는다.

## 처리 규칙

1. 캠페인 접근 권한과 조회 시간 구간을 검증한다.
2. 집계 Read Model을 기준 시각과 함께 조회한다.
3. count·amount 집계와 `asOf`를 반환한다.

## 상태 변경과 트랜잭션

- 상태를 변경하지 않는다.
- 집계 지연을 0으로 위조하지 않고 `asOf`와 lag를 제공한다.

## 멱등성과 동시성

- Query이므로 멱등 키가 필요 없다.
- Event ID unique와 집계 bucket key로 중복 반영을 막는다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 권한 없음·캠페인 없음 | 허용 범위 밖 상세를 숨김 | 접근 권한을 확인한다. |
| projection 장애 | 0 집계로 반환하지 않음 | 복구 뒤 다시 조회한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Service | 캠페인 성과 Query |
| Read Model | `CouponIssuanceAndRedemptionPerformance` |
| 원천 Event | 발급·사용·회수·대량 집계 Event |

## 관측성과 운영

- 조회 구간, 결과 bucket 수, projection lag를 기록한다.

## 검증 항목

- 집계가 append-only 원장과 샘플 대조된다.
- 판매자 workload가 운영 전체 집계를 조회하지 못한다.
- projection 지연이 빈 성공으로 보이지 않는다.

## 연관 시퀀스

- 운영·마케팅 화면 시퀀스는 아직 없다.

## 호환성과 변경 정책

- 집계 dimension 추가는 권한과 cardinality 검토 뒤 선택 필드로 추가한다.

## 확인 필요

- `HOTSPOT.A.19-07`: 판매자 전용 성과 Endpoint와 집계 범위.
