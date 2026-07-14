---
id: API.A.200.TRACEABILITY
title: 판매자 PAGE·API·도메인·MSA 추적성 매트릭스
type: service-design-traceability-matrix
status: draft
tags: [service-design, seller, page, api, command, read-model, msa]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
api_design: SD.A.20040
page_ids: [PAGE.A.200, PAGE.A.201, PAGE.A.202, PAGE.A.203, PAGE.A.204, PAGE.A.205, PAGE.A.206, PAGE.A.207, PAGE.A.208, PAGE.A.209, PAGE.A.210, PAGE.A.211]
---

# 판매자 PAGE·API·도메인·MSA 추적성 매트릭스

현재 page kind와 command path는 `dropmong-web` BFF 코드의 식별자일 뿐 목표 API가 아니다. 목표 구조에서는 브라우저가 Kubernetes Ingress를 통해 표의 실제 소유 서비스 API를 호출한다.

| PAGE | 현행 BFF 코드 식별자 | 논리 API | CMD·RM | 목표 소유 서비스·원천 | 상태 |
| --- | --- | --- | --- | --- | --- |
| `PAGE.A.200` | `dashboard` | `API.A.200-17` | `RM.A.200-05` | 소유 미확정; Catalog·Order·Payment·Coupon Event 필요 | 제공 불가 |
| `PAGE.A.201` | `drops` | `API.A.200-18` | `RM.A.200-06` | `catalog-service` 후보 | seller 조회·공개 상태 Event 미구현 |
| `PAGE.A.202` | `products`, `products/save` | `API.A.200-09~10` | `RM.A.200-03`, `CMD.A.200-06` | `catalog-service` 후보 | seller 상품 계약 미구현 |
| `PAGE.A.203` | `drop-editor`, `drop-drafts/save`, `reviews/submit` | `API.A.200-11~13` | `RM.A.200-04`, `CMD.A.200-07~08` | `catalog-service` 후보 | seller 제안 계약 미구현 |
| `PAGE.A.204` | `review`, `reviews/submit` | `API.A.200-11`, `API.A.200-13~15` | `RM.A.200-04`, `CMD.A.200-08~10` | `catalog-service` 후보 + 플랫폼 검수 | decision 계약 대기 |
| `PAGE.A.205` | `orders`, `order-exports/create` | `API.A.200-19~22` | `RM.A.200-07`, `CMD.A.200-12~14` | `order-service` 후보 | seller 주문·export 계약 미구현 |
| `PAGE.A.206` | `coupons`, `coupons/save` | Coupon `API.A.19-10~13/16` | Coupon `CMD.A.19-01~04`, `RM.A.19-03~04` | `coupon-service` | 일부 구현; seller 공개·목록·제휴 계약 대기 |
| `PAGE.A.207` | `analytics` | `API.A.200-23`, Coupon `API.A.19-16/25` | `RM.A.200-08` | 통합 조회 소유 미확정 | 제공 불가 |
| `PAGE.A.208` | `settlements` | `API.A.200-24`, Coupon `API.A.19-25` 참조 | `RM.A.200-09` | 정산 소유 서비스 미확정 | 제공 불가 |
| `PAGE.A.209` | `store`, `account/save`, `store-profile/save` | `API.A.200-02~04` | `RM.A.200-01`, `CMD.A.200-01~02` | Seller Management 소유 미확정 | 제공 불가 |
| `PAGE.A.210` | `members`, `members/invite`, `roles/permissions/save` | `API.A.200-01`, `API.A.200-05~08`, Auth `API.A.300-17/29` | `RM.A.200-02`, `CMD.A.200-03~05` | membership 소유 미확정 + `auth-service` proof | 제공 불가 |
| `PAGE.A.211` | `issues`, `issues/create` | `API.A.200-25~28` | `RM.A.200-10`, `CMD.A.200-15~17` | Seller Issue·플랫폼 운영 case 소유 미확정 | 제공 불가 |

## 목표 호출 규칙

- 브라우저는 Ingress가 공개한 실제 소유 서비스 operation만 호출한다. 현행 `commandPath`를 downstream path나 API ID로 전달하지 않는다.
- 페이지 하나가 여러 서비스의 독립 기능을 배치할 수는 있지만 브라우저가 원천 값을 합쳐 업무 지표를 만들지 않는다.
- dashboard·analytics·settlements는 집계 조회 모델 소유자가 정해질 때까지 성공 모양의 빈 값이나 임시 fan-out으로 제공하지 않는다.
- 수신 서비스는 검증된 인증 context를 받은 뒤 현재 membership, permission, seller scope와 resource version을 다시 확인한다.
- 현행 fixture와 `/api/web/seller/**`는 제거 대상 코드이며 목표 추적성의 연결점이 아니다.
