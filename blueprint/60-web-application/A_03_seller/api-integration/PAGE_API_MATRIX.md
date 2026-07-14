---
id: seller-web-page-api-matrix
title: 판매자 페이지별 API 연동 매트릭스
type: web-api-integration-matrix
status: draft
tags: [web, seller, page, api, read-model, command, ingress]
source: local
created: 2026-07-13
updated: 2026-07-13
page_ids: [PAGE.A.200, PAGE.A.201, PAGE.A.202, PAGE.A.203, PAGE.A.204, PAGE.A.205, PAGE.A.206, PAGE.A.207, PAGE.A.208, PAGE.A.209, PAGE.A.210, PAGE.A.211]
---

# 판매자 페이지별 API 연동 매트릭스

| PAGE | 화면 목적 | CMD·RM | 현행 BFF 식별자 | 목표 MSA·API | 상태 |
| --- | --- | --- | --- | --- | --- |
| `PAGE.A.200` | 대시보드 | `RM.A.200-05` | `dashboard` | 소유 미확정, `API.A.200-17` 논리 초안 | 제공 불가 |
| `PAGE.A.201` | 드롭 관리 | `RM.A.200-06` | `drops` | `catalog-service` 후보, `API.A.200-18` | seller 계약 미구현 |
| `PAGE.A.202` | 상품 관리 | `RM.A.200-03`, `CMD.A.200-06` | `products`, `products/save` | `catalog-service` 후보, `API.A.200-09~10` | seller 계약 미구현 |
| `PAGE.A.203` | 드롭 초안 | `RM.A.200-04`, `CMD.A.200-07~08` | `drop-editor`, `drop-drafts/save`, `reviews/submit` | `catalog-service` 후보, `API.A.200-11~13` | seller 계약 미구현 |
| `PAGE.A.204` | 검수·변경 | `RM.A.200-04`, `CMD.A.200-08~10` | `review`, `reviews/submit` | `catalog-service` 후보, `API.A.200-11/13~15` | 검수 decision 계약 대기 |
| `PAGE.A.205` | 주문·출고 자료 | `RM.A.200-07`, `CMD.A.200-12~14` | `orders`, `order-exports/create` | `order-service` 후보, `API.A.200-19~22` | seller 주문·export 미구현 |
| `PAGE.A.206` | 쿠폰·제휴 | Coupon `RM.A.19-03~04`, `CMD.A.19-01~04` | `coupons`, `coupons/save` | `coupon-service`, `API.A.19-10~13/16` | 일부 구현, seller 공개·목록·제휴 보강 대기 |
| `PAGE.A.207` | 판매 분석 | `RM.A.200-08` | `analytics` | 소유 미확정, `API.A.200-23` | 제공 불가 |
| `PAGE.A.208` | 정산 조회 | `RM.A.200-09` | `settlements` | 소유 미확정, `API.A.200-24` | 제공 불가 |
| `PAGE.A.209` | 판매자·스토어 설정 | `RM.A.200-01`, `CMD.A.200-01~02` | `store`, `account/save`, `store-profile/save`, `onboarding` | 소유 미확정, `API.A.200-02~04` | 제공 불가 |
| `PAGE.A.210` | 팀·권한 | `RM.A.200-02`, `CMD.A.200-03~05` | `members`, `members/invite`, `roles/permissions/save` | 소유 미확정 + Auth `API.A.300-17/29` | 제공 불가 |
| `PAGE.A.211` | 운영 이슈 | `RM.A.200-10`, `CMD.A.200-15~17` | `issues`, `issues/create` | 소유 미확정, `API.A.200-25~28` | 제공 불가 |

## 공통 선행 계약

| 계약 | 소유자 | 필요한 결정 |
| --- | --- | --- |
| 인증 context | `auth-service` + Ingress | `user_id`, Session ref 전달과 외부 header 제거 방식 |
| seller membership | 미확정 | 원장 서비스, 다중 membership, permission version |
| seller resource 검증 | 각 업무 서비스 | membership 재검증과 resource seller ownership 확인 |
| 오류 | 각 업무 서비스 | `403`, 동일 `404`, `409`, typed `503`의 service namespace code |
| 최신성 | 조회 모델 소유 서비스 | `asOf`, watermark, stale, partial, unavailable section |
| mutation | 각 업무 서비스 | `Idempotency-Key`, `If-Match`, 재시도와 감사 |
| 강한 재인증 | `auth-service` + 업무 서비스 | seller purpose proof와 최신 권한 재확인 |

## 전환 단위

1. Ingress와 membership 계약이 확정될 때까지 현재 보호 seller API를 운영 공개하지 않는다.
2. Catalog·Order·Coupon처럼 소유자가 명확한 operation을 서비스별로 구현하고 해당 PAGE의 fixture만 제거한다.
3. 통합 조회 소유자가 생기기 전에는 dashboard·analytics·settlement를 브라우저 fan-out으로 대체하지 않는다.
4. 각 전환은 Ingress 보안, consumer contract, typed 오류, buyer 회귀 E2E를 함께 검증한다.
