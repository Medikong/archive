---
id: API.A.19.SELLER_GAPS
title: Coupon 판매자 연동 계약 gap
type: service-design-contract-gap
status: draft
tags: [service-design, coupon, seller, contract-gap]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.19
api_design: SD.A.1940
---

# Coupon 판매자 연동 계약 gap

## 책임 경계

Context 쿠폰은 캠페인·발급·사용·비용 귀속을 소유하고, Context 판매자는 seller membership·permission과 판매자 포털 Read Model을 소유한다. Coupon은 signed seller scope만 신뢰하지 않고 현재 membership·permission version을 Context 판매자에서 재검증해야 한다.

## API별 보강

| API | 현재 계약 | 필요한 seller 계약 | 상태 |
| --- | --- | --- | --- |
| `API.A.19-10` | 캠페인 등록, seller issuer 후보 | seller 소유 상품·drop snapshot, 위험 기반 approval, seller permission 재검증 | 설계 보강·구현 대기 |
| `API.A.19-11` | 발급 수량·기간 설정 | 같은 seller 소유 campaign과 current policy version 검증 | 설계 보강·구현 대기 |
| `API.A.19-12` | 검토 workload 결정 | 판매자 직접 호출 금지, seller 포털에는 결과 projection만 제공 | 기존 책임 유지 |
| `API.A.19-13` | 새 policy version 등록 | seller 허용 변경과 재검수 필요 변경 분리, ETag/version adapter | 설계 보강·구현 대기 |
| `API.A.19-16` | seller/운영 성과 Query | seller scope, `groupBy`, 자기 소유·비용 부담 범위, stale/partial metadata | archive 설계됨, 구현 후속 |
| `API.A.19-25` | 정산·운영 비용 귀속 Query | seller 직접 전체 조회 금지. Seller Operations projection에는 seller 귀속 ref만 전달 | 외부 adapter 계약 대기 |

## 누락 operation

- seller별 campaign 목록·상태·발급 대기 표시 Query
- 플랫폼 제휴 제안 목록·상세와 판매자 수락·거절 Command
- seller scope 재검증용 Context 판매자 API 호출 또는 versioned membership snapshot contract

이 operation들은 Coupon·플랫폼 제휴 소유권과 상태 전이가 정해지기 전까지 `API.A.19-*` 번호나 성공 schema를 만들지 않는다. 판매자 BFF의 `coupons/save`를 canonical Coupon API로 사용하지 않는다.

## 연결

- [Context 판매자 설계](../../A_200_seller/README.md)
- [판매자 PAGE/API 매트릭스](../../A_200_seller/A_200_40-api/page-api-traceability.md)
