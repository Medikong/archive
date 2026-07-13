---
id: requirements-index
title: 요구사항 인덱스
type: requirements-index
status: draft
tags: [requirements, index, dropmong]
source: local
created: 2026-07-06
updated: 2026-07-10
---

# 요구사항 인덱스

## 역할

DropMong MVP A의 기능 요구사항, 비기능 요구사항, 페인포인트 개선 근거를 모아 보는 인덱스다.

## 템플릿

- [요구사항 템플릿](.template/requirements.md)

## 실제 문서

- [REQ.A.01 한정 상품 드롭 커머스](REQ_A_01_limited_drop_commerce.md)
- [REQ.A.02 쿠폰/혜택](REQ_A_02_coupon_benefit.md)
- [REQ.A.03 판매자](REQ_A_03_seller.md)
- [REQ.A.04 플랫폼 운영자 어드민](REQ_A_04_platform_operator_admin.md)
- [REQ.A.05 인증 및 회원](REQ_A_05_auth_member.md)
- [REQ.A.06 쿠버네티스 클러스터 아키텍처](REQ_A_06_kubernetes_cluster_architecture.md)
- [REQ.A.07 관심·랭킹](REQ_A_07_interest_ranking.md)
- [REQ.A.08 반응형 웹 애플리케이션](REQ_A_08_web_application.md)

## 예시 문서

- [REQ.A.01 주문 결제 예시](.example/REQ_A_01_order_checkout.md)

## 연관 태그

🏷️ 페이지 참조: [PAGE.A.01](../10-sitemap/buyer-mobile-web/PAGE_A_01_homepage.md), [PAGE.A.02](../10-sitemap/buyer-mobile-web/PAGE_A_02_product_detail.md), [PAGE.A.09](../10-sitemap/buyer-mobile-web/PAGE_A_09_waiting_products.md), [PAGE.A.06](../10-sitemap/buyer-mobile-web/PAGE_A_06_shopping_cart.md), [PAGE.A.10](../10-sitemap/buyer-mobile-web/PAGE_A_10_my.md), [PAGE.A.11](../10-sitemap/buyer-mobile-web/PAGE_A_11_payment.md), [PAGE.A.14](../10-sitemap/buyer-mobile-web/PAGE_A_14_order_complete.md), [PAGE.A.15](../10-sitemap/buyer-mobile-web/PAGE_A_15_order_history.md), [PAGE.A.16](../10-sitemap/buyer-mobile-web/PAGE_A_16_track_order.md), [PAGE.A.17](../10-sitemap/buyer-mobile-web/PAGE_A_17_shipping_order_manage.md), [PAGE.A.19](../10-sitemap/buyer-mobile-web/PAGE_A_19_coupon_wallet/PAGE_A_19_owned_coupon.md), [PAGE.A.22](../10-sitemap/buyer-mobile-web/PAGE_A_22_wishlist.md), [PAGE.A.23](../10-sitemap/buyer-mobile-web/PAGE_A_23_trending_products.md), [PAGE.A.200](../10-sitemap/PAGE_A_200_seller_portal/README.md), [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md), [PAGE.A.310](../10-sitemap/PAGE_A_310_password_find/PAGE_A_310_password_find.md) | UI 참조: [UI.A.01](../20-ui/buyer-mobile-web/UI_A_01_homepage.md), [UI.A.02](../20-ui/buyer-mobile-web/UI_A_02_product_detail.md), [UI.A.09](../20-ui/buyer-mobile-web/UI_A_09_waiting_products.md), [UI.A.06](../20-ui/buyer-mobile-web/UI_A_06_shopping_cart.md), [UI.A.10](../20-ui/buyer-mobile-web/UI_A_10_my.md), [UI.A.11](../20-ui/buyer-mobile-web/UI_A_11_payment.md), [UI.A.14](../20-ui/buyer-mobile-web/UI_A_14_order_complete.md), [UI.A.15](../20-ui/buyer-mobile-web/UI_A_15_order_history.md), [UI.A.16](../20-ui/buyer-mobile-web/UI_A_16_track_order.md), [UI.A.17](../20-ui/buyer-mobile-web/UI_A_17_shipping_order_manage.md), [UI.A.19](../20-ui/buyer-mobile-web/UI_A_19_coupon_wallet/UI_A_19_coupon_wallet.md), [UI.A.22](../20-ui/buyer-mobile-web/UI_A_22_wishlist.md), [UI.A.23](../20-ui/buyer-mobile-web/UI_A_23_trending_products.md), [UI.A.200~211](../20-ui/UI_A_200_seller_portal/README.md), [UI.A.300](../20-ui/UI_A_300_auth_member/UI_A_300_auth_member.md), [UI.A.310](../20-ui/UI_A_310_password_find/UI_A_310_password_find.md) | 유스케이스 참조: [UC.A.01](../30-uc/UC_A_01_buyer_purchase_delivery.md), [UC.A.02](../30-uc/UC_A_02_seller_manage_drop.md), [UC.A.03](../30-uc/UC_A_03_platform_operator_admin.md), [UC.A.04](../30-uc/UC_A_04_cs_order_coupon_support.md), [UC.A.07](../30-uc/UC_A_07_interest_ranking.md), [UC.A.19](../30-uc/UC_A_19_coupon_wallet.md), [UC.A.300](../30-uc/UC_A_300_auth_member.md)

## 확인 필요

- `REQ.A.01`의 구매자/판매자/플랫폼 운영자 용어를 최신 액터 정의에 맞춰 한 번 더 정리한다.
- 쿠폰, 판매자, 플랫폼 운영자 요구사항이 서비스/API 문서로 내려갈 때 세부 ID 연결을 갱신한다.
- 인증 및 회원 요구사항이 서비스/API 문서로 내려갈 때 세션 기본값, 가상 SMS 인증번호 정책, 로그인 실패 잠금 정책, 인증 수단 연동 상태 전이를 갱신한다.
