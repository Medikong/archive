---
id: buyer-web-page-api-matrix
title: 구매자 페이지별 API 연동 매트릭스
type: web-api-integration-matrix
status: draft
tags: [web, buyer, page, api, bff, ingress]
source: local
created: 2026-07-16
updated: 2026-07-16
page_ids: [PAGE.A.01, PAGE.A.02, PAGE.A.06, PAGE.A.09, PAGE.A.10, PAGE.A.11, PAGE.A.14, PAGE.A.15, PAGE.A.16, PAGE.A.17, PAGE.A.19, PAGE.A.22, PAGE.A.23]
---

# 구매자 페이지별 API 연동 매트릭스

`PAGE 기준 route`는 사이트맵의 목표다. `현재 코드 route`가 `없음`이면 PAGE·UI가 없다는 뜻이 아니라 `dropmong-web` 구현이 아직 없다는 뜻이다. GitOps·Ingress 상태는 서비스별 차이가 있어 [서비스 API 인벤토리](SERVICE_API_INVENTORY.md)에서 관리하며, 선언이 있어도 이 표의 프론트 연결 상태를 바꾸지 않는다.

| PAGE | PAGE 기준 route | 현재 코드 route | 현재 함수·Route Handler | 목표 소유 서비스·API | 서비스 API | 프론트 | 로컬·검증 | mock·fixture / 다음 작업 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [A.01](../../../10-sitemap/buyer-mobile-web/PAGE_A_01_homepage.md) | `/` | `/` | `getHomePage`; `GET /api/web/home` | Catalog 필수, Auth·Interest·Notification 선택 | Catalog 구현, 부가 API 일부 구현 | Catalog 코드 지원 | mock만 | `developmentDrops`; Catalog 실연동 1순위 |
| [A.02](../../../10-sitemap/buyer-mobile-web/PAGE_A_02_product_detail.md) | `/products/{productId}` | `/products/[productId]` | `getProductDetailPage`; `GET /api/web/products/[productId]` | Catalog 필수, Interest 선택 | Catalog·Interest 구현 | Catalog 코드 지원 | mock만 | Catalog mock, 개인화 문구; Catalog 뒤 Auth·Interest |
| [A.06](../../../10-sitemap/buyer-mobile-web/PAGE_A_06_shopping_cart.md) | `/cart` | 없음 | 없음 | 장바구니 소유 미확정 | 없음 | 미연결 | 없음 | route·원장 소유자 결정 필요 |
| [A.09](../../../10-sitemap/buyer-mobile-web/PAGE_A_09_waiting_products.md) | `/rankings/waiting` | 없음 | 없음 | `interest-service` 예정 랭킹 | `GET /v1/rankings/drops/upcoming` 구현 | 미연결 | 없음 | route와 buyer client 추가 |
| [A.10](../../../10-sitemap/buyer-mobile-web/PAGE_A_10_my.md) | `/my` | 없음 | 없음 | User 필수, Order·Coupon·Interest·Notification section | API 일부 구현, Order 목록 없음 | 미연결 | 없음 | section별 계약·부분 실패 결정 |
| [A.11](../../../10-sitemap/buyer-mobile-web/PAGE_A_11_payment.md) | `/checkout` | `/checkout` | `getCheckoutSnapshot`, `confirmCheckout`; checkout GET/POST handlers | Checkout 소유 미확정 | canonical API 없음 | fixture | mock만 | 배송지·카드·쿠폰·포인트·총액 fixture; 계약 확정 3순위 |
| [A.14](../../../10-sitemap/buyer-mobile-web/PAGE_A_14_order_complete.md) | `/orders/complete` | `/orders/complete` | `getOrderResult`; `GET /api/web/orders/[orderId]` polling | Order·Payment 상태 계약 | 두 서비스 단건 API 구현 | fixture | mock만 | `dev-order.*`, 배송 예정 fixture; 상태 연결 4순위 |
| [A.15](../../../10-sitemap/buyer-mobile-web/PAGE_A_15_order_history.md) | `/orders` | 없음 | 없음 | `order-service` buyer 목록 | 목록 API 없음 | 미연결 | 없음 | 목록·paging·소유권 계약 필요 |
| [A.16](../../../10-sitemap/buyer-mobile-web/PAGE_A_16_track_order.md) | `/orders/:orderId/tracking` | 없음 | 없음 | 배송 소유 미확정 | canonical API 없음 | 미연결 | 없음 | 배송 원장·조회 계약 필요 |
| [A.17](../../../10-sitemap/buyer-mobile-web/PAGE_A_17_shipping_order_manage.md) | `/orders/:orderId/manage` | 없음 | 없음 | Order + 배송 소유 미확정 | 관리 API 없음 | 미연결 | 없음 | 취소·변경·배송 작업 소유권 필요 |
| [A.19](../../../10-sitemap/buyer-mobile-web/PAGE_A_19_coupon_wallet/PAGE_A_19_owned_coupon.md) | `/my/coupons` | 없음 | 없음 | `coupon-service` | 보유 쿠폰 목록·상세 구현 | 미연결 | 없음 | route·Auth·client 연결 5순위 |
| [A.22](../../../10-sitemap/buyer-mobile-web/PAGE_A_22_wishlist.md) | `/my/wishlist` | 없음 | 없음 | `interest-service` | 관심 목록·변경 구현 | 미연결 | 없음 | route·Auth·client 연결 5순위 |
| [A.23](../../../10-sitemap/buyer-mobile-web/PAGE_A_23_trending_products.md) | `/rankings/trending` | 없음 | 없음 | `interest-service` | `GET /v1/rankings/drops/trending` 구현 | 미연결 | 없음 | route와 buyer client 추가 5순위 |

## 공통 선행 조건

| 계약 | 현재 상태 | 다음 결정 |
| --- | --- | --- |
| Catalog 실행 설정 | 코드 지원, 로컬·CI·GitOps 미연결 | 실제 URL과 실연동 테스트 |
| Auth browser session | service API 구현, 웹은 개발 cookie | opaque session context 정합화 |
| Checkout | fixture, 소유 미확정 | snapshot·confirm 원장과 오류·멱등 계약 |
| Order·Payment 상태 | 서비스 단건 API, 웹 미연결 | Checkout 결과와 상태 조회 식별자 |
| 배송·포인트·결제수단 | 소유 미확정 | 원장과 canonical API |
| DropMong 배포 | Web·Catalog·Order·Interest는 선언 없음, User·Coupon은 일부 선언 있음, 동명 Auth·Payment·Notification은 동일성 미확인 | 서비스별 identity, route 포함 범위, 실제 동기화와 필요한 환경 변수 |

## 전환 판정

PAGE를 `실연결`로 바꾸려면 다음을 모두 확인한다.

1. 소유 서비스 API와 테스트가 존재한다.
2. `dropmong-web` 함수 또는 browser client가 canonical API를 호출한다.
3. 로컬 또는 통합 환경에서 mock 없이 성공·오류를 검증한다.
4. 배포 대상이면 GitOps 환경 변수, service DNS와 Ingress 경계가 선언된다.
5. fixture와 성공 모양 fallback이 실제 경로에서 제거된다.
