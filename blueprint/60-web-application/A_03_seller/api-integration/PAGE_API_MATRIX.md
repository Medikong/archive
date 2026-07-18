---
id: seller-web-page-api-matrix
title: 판매자 페이지별 API 연동 매트릭스
type: web-api-integration-matrix
status: draft
tags: [web, seller, page, api, read-model, command, ingress]
source: local
created: 2026-07-13
updated: 2026-07-16
page_ids: [PAGE.A.200, PAGE.A.201, PAGE.A.202, PAGE.A.203, PAGE.A.204, PAGE.A.205, PAGE.A.206, PAGE.A.207, PAGE.A.208, PAGE.A.209, PAGE.A.210, PAGE.A.211]
---

# 판매자 페이지별 API 연동 매트릭스

현재 모든 PAGE는 `SellerWorkspacePage(kind)` → `readSellerPage` → `getSellerPageFixture` 경로를 사용한다. Command는 `/api/web/seller/{commandPath}`와 fixture 결과를 사용한다. 아래의 `프론트` 열에서 `fixture`는 실제 서비스 미연결을 뜻한다.

| PAGE | 실제 route | 현재 함수·식별자 | 목표 소유 서비스·API | API 구현 | seller 사용 | Ingress | 프론트 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `PAGE.A.200` | `/seller` | `dashboard` | 소유 미확정, `API.A.200-17` | 없음 | 불가 | 미연결 | fixture |
| `PAGE.A.201` | `/seller/drops` | `drops` | `catalog-service` 후보, `API.A.200-18` | buyer 공개 조회만 | 불가 | 미연결 | fixture |
| `PAGE.A.202` | `/seller/products` | `products`, `products/save` | `catalog-service` 후보, `API.A.200-09~10` | seller API 없음 | 불가 | 미연결 | fixture |
| `PAGE.A.203` | `/seller/drops/new`, `/seller/drops/[dropId]/edit` | `drop-editor`, `drop-drafts/save` | `catalog-service` 후보, `API.A.200-11~13` | seller API 없음 | 불가 | 미연결 | fixture |
| `PAGE.A.204` | `/seller/drops/[dropId]/review` | `review`, `reviews/submit` | `catalog-service` 후보, `API.A.200-11/13~15` | seller API 없음 | 불가 | 미연결 | fixture |
| `PAGE.A.205` | `/seller/orders` | `orders`, `order-exports/create` | `order-service` 후보, `API.A.200-19~22` | buyer 주문만 | 불가 | 미연결 | fixture |
| `PAGE.A.206` | `/seller/coupons` | `coupons`, `coupons/save` | `coupon-service`, `API.A.19-10~13/16` | internal 일부 | 공개·scope 부족 | 미연결 | fixture |
| `PAGE.A.207` | `/seller/analytics` | `analytics` | 소유 미확정, `API.A.200-23` | 원천 API 단편만 | 불가 | 미연결 | fixture |
| `PAGE.A.208` | `/seller/settlements` | `settlements` | 소유 미확정, `API.A.200-24` | settlement 없음 | 불가 | 미연결 | fixture |
| `PAGE.A.209` | `/seller/settings/store` | `store`, `account/save`, `store-profile/save`, `onboarding` | 소유 미확정, `API.A.200-02~04` | seller 설정 없음 | 불가 | 미연결 | fixture |
| `PAGE.A.210` | `/seller/settings/members` | `members`, `members/invite`, `roles/permissions/save` | 소유 미확정, `API.A.200-05~08`; Auth 재인증 보조 후보 | 일반 Auth proof만 일부 구현, seller purpose 없음 | 불가 | 미연결 | fixture |
| `PAGE.A.211` | `/seller/issues` | `issues`, `issues/create` | 소유 미확정, `API.A.200-25~28` | 없음 | 불가 | 미연결 | fixture |

## 공통 선행 계약

| 계약 | 소유자 | 현재 상태 | 필요한 결정 |
| --- | --- | --- | --- |
| 인증 context | `auth-service` + Ingress | Auth API 구현, 웹 미연결·DropMong Ingress 동일성 미확인 | browser session과 외부 header 제거 방식 |
| seller membership | 미확정 | API·Ingress·프론트 없음 | 원장 서비스, 다중 membership, permission version |
| seller resource 검증 | 각 업무 서비스 | seller API 없음 | membership 재검증과 resource seller ownership |
| 오류 | 각 업무 서비스 | 논리 초안 | 동일 `404`, `403`, `409`, typed `503` code |
| 최신성 | 조회 모델 소유 서비스 | 소유 미확정 | `asOf`, watermark, stale, partial |
| mutation | 각 업무 서비스 | seller API 없음 | 멱등키, version, 재시도와 감사 |
| 강한 재인증 | `auth-service` + 업무 서비스 | `link_identity`·`replace_phone` proof와 `purchase` resume만 구현. seller purpose·소비·Ingress 없음 | seller purpose 검증과 최신 권한 재확인 |

## 전환 조건

1. API가 코드·OpenAPI·테스트에 구현된다.
2. seller principal, membership, resource ownership, 마스킹·감사 조건을 충족한다.
3. DropMong GitOps에 Ingress route와 보안 정책이 선언된다.
4. PAGE가 canonical API를 호출하고 fixture와 현행 Seller BFF 의존을 제거한다.

네 조건 중 하나라도 충족하지 않으면 `실연결`로 표시하지 않는다. 통합 조회 소유자가 생기기 전에는 dashboard·analytics·settlement를 브라우저나 Server Component fan-out으로 대체하지 않는다.
