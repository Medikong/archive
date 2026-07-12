---
id: API.A.19-25
title: 쿠폰 비용 귀속 조회 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, settlement, cost, query]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-25 쿠폰 비용 귀속 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/internal/coupon-cost-attributions` |
| operationId | `listCouponCostAttributions` |
| 역할 | 주문·캠페인별 할인 비용 귀속과 회수 보정을 조회한다. |
| API 유형 | Query |
| 인증·권한 | 정산·운영 workload와 비용 조회 권한 |
| 노출 범위 | internal |
| 멱등성 | 해당 없음 |
| 캐시 | `no-store` |
| 호환성 | cursor pagination |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_25_list_coupon_cost_attributions.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-024`, `UC.A.19-22` |
| BC | `EVT.A.19-28`, `RM.A.19-08` |
| 도메인 | [사용](../A_19_10-domain-model/redemption.md) |

## 책임과 경계

- 주문·redemption·campaign ref, 할인 금액과 플랫폼·판매자·공동·보상 비용 배분을 제공한다.
- 정산 예정·보류·확정과 지급 상태를 소유하거나 반환하지 않는다.
- 회수 Event는 원본 귀속을 수정하지 않고 보정 항목으로 제공한다.

## 보안과 개인정보

- 정산 workload와 조회 범위를 확인한다.
- 사용자 프로필·결제수단·주문 상세를 반환하지 않는다.
- 금액 집계와 외부 ref를 로그·metric label에 넣지 않는다.

## 처리 규칙

1. order/campaign/time range filter, cursor와 권한을 검증한다.
2. 비용 귀속 append-only Read Model을 조회한다.
3. 원본 귀속과 보정 항목, 통화, `asOf`를 반환한다.

## 상태 변경과 트랜잭션

- 상태를 변경하지 않는다.
- 조회 결과를 정산 확정 상태로 간주하지 않는다.

## 멱등성과 동시성

- Query이므로 멱등 키가 필요 없다.
- `(occurred_at, attribution_id)` cursor와 Event ID unique로 안정적인 목록을 제공한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 통화·범위 입력 오류 | 내부 원장 구조를 숨김 | 유효한 filter로 다시 조회한다. |
| projection 지연·장애 | 0 비용으로 위조하지 않음 | `asOf`를 확인하고 재조회한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Service | 비용 귀속 Query |
| Read Model | `CouponCostAttribution` |
| 원천 Event | `EVT.A.19-22`, `24`, `28` |
| 외부 경계 | 정산 시스템은 회계 상태를 별도 소유 |

## 관측성과 운영

- 조회 구간, 결과 수, 통화 종류와 projection lag를 기록한다.

## 검증 항목

- 귀속 금액 합계가 할인 금액과 일치한다.
- 회수가 원본 행 삭제가 아니라 보정 행으로 나타난다.
- 정산 상태와 개인정보가 응답에 섞이지 않는다.

## 연관 시퀀스

- 사용 확정·회수 Event 뒤 정산 시스템 조회·소비에 연결된다.

## 호환성과 변경 정책

- 새 비용 주체 type은 기존 소비자의 unknown 처리와 회계 매핑 검토 뒤 추가한다.

## 확인 필요

- 정산 담당자 전용 UI와 Event 소비·Query 사용 비율은 아직 정하지 않았다.
