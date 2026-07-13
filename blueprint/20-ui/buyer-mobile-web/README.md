---
id: ui-buyer-mobile-web
title: 구매자 모바일 웹앱 UI 인덱스
type: ui-client-index
status: draft
tags: [ui, buyer, mobile-web, responsive-web, dropmong]
source: local
created: 2026-07-10
updated: 2026-07-10
---

# 구매자 모바일 웹앱 UI 인덱스

## 문서 역할

구매자 모바일 웹앱의 PAGE 문서, 화면 시안, 컴포넌트 근거를 연결한다. 기존 시안의 DropMong 보라색 계열, 흰색 카드, 짙은 남색 본문, 한정 판매 배지와 캐릭터 사용법을 유지하면서 브라우저 기반 반응형 화면으로 구체화한다.

## UI 문서

| UI ID | 문서 | PAGE | 신규 모바일 웹 시안 |
| --- | --- | --- | --- |
| [UI.A.01](UI_A_01_homepage.md) | 홈 | [PAGE.A.01](../../10-sitemap/buyer-mobile-web/PAGE_A_01_homepage.md) | [시안](assets/UI_A_01_homepage/UI_A_01_10_buyer_mobile_web.png) |
| [UI.A.02](UI_A_02_product_detail.md) | 상품 상세 | [PAGE.A.02](../../10-sitemap/buyer-mobile-web/PAGE_A_02_product_detail.md) | [시안](assets/UI_A_02_product_detail/UI_A_02_10_buyer_mobile_web.png) |
| [UI.A.06](UI_A_06_shopping_cart.md) | 장바구니 | [PAGE.A.06](../../10-sitemap/buyer-mobile-web/PAGE_A_06_shopping_cart.md) | [시안](assets/UI_A_06_shopping_cart/UI_A_06_10_buyer_mobile_web.png) |
| [UI.A.10](UI_A_10_my.md) | 마이 | [PAGE.A.10](../../10-sitemap/buyer-mobile-web/PAGE_A_10_my.md) | [시안](assets/UI_A_10_my/UI_A_10_10_buyer_mobile_web.png) |
| [UI.A.11](UI_A_11_payment.md) | 주문·결제 | [PAGE.A.11](../../10-sitemap/buyer-mobile-web/PAGE_A_11_payment.md) | [시안](assets/UI_A_11_payment/UI_A_11_10_buyer_mobile_web.png) |
| [UI.A.14](UI_A_14_order_complete.md) | 주문 완료 | [PAGE.A.14](../../10-sitemap/buyer-mobile-web/PAGE_A_14_order_complete.md) | [시안](assets/UI_A_14_order_complete/UI_A_14_10_buyer_mobile_web.png) |
| [UI.A.15](UI_A_15_order_history.md) | 주문 내역 | [PAGE.A.15](../../10-sitemap/buyer-mobile-web/PAGE_A_15_order_history.md) | [시안](assets/UI_A_15_order_history/UI_A_15_10_buyer_mobile_web.png) |
| [UI.A.16](UI_A_16_track_order.md) | 배송 조회 | [PAGE.A.16](../../10-sitemap/buyer-mobile-web/PAGE_A_16_track_order.md) | [시안](assets/UI_A_16_track_order/UI_A_16_10_buyer_mobile_web.png) |
| [UI.A.17](UI_A_17_shipping_order_manage.md) | 배송·주문 관리 | [PAGE.A.17](../../10-sitemap/buyer-mobile-web/PAGE_A_17_shipping_order_manage.md) | [시안](assets/UI_A_17_shipping_order_manage/UI_A_17_10_buyer_mobile_web.png) |
| [UI.A.19](UI_A_19_coupon_wallet/UI_A_19_coupon_wallet.md) | 보유 쿠폰·코드 등록 | [PAGE.A.19](../../10-sitemap/buyer-mobile-web/PAGE_A_19_coupon_wallet/PAGE_A_19_owned_coupon.md) | [시안](assets/UI_A_19_coupon_wallet/UI_A_19_10_buyer_mobile_web.png) |
| [UI.A.22](UI_A_22_wishlist.md) | 찜리스트 | [PAGE.A.22](../../10-sitemap/buyer-mobile-web/PAGE_A_22_wishlist.md) | [시안](assets/UI_A_22_wishlist/UI_A_22_10_buyer_mobile_web.png) |

## 시각 기준

| 항목 | 기준 |
| --- | --- |
| 브랜드 | 기존 `DropMong` 워드마크와 보라색 강조색을 사용한다. |
| 컬러 | 본문은 짙은 남색, 표면은 흰색, 주요 CTA는 선명한 보라색, 오류·성공·경고는 의미가 겹치지 않게 분리한다. |
| 글자 | 본문 최소 `16px`, 숫자·가격·남은 시간은 자릿수를 빠르게 읽을 수 있게 강조한다. |
| 조작 | 터치 영역 최소 `44×44px`, 키보드 포커스 표시, 버튼의 활성·비활성·진행 중 상태를 구분한다. |
| 반응형 | `375`, `768`, `1024`, `1440px`에서 가로 스크롤 없이 같은 업무를 수행할 수 있어야 한다. |
| 모션 | 카운트다운·상태 갱신에만 절제해 사용하고 `prefers-reduced-motion`을 따른다. |

## 시안 작성 원칙

- 이미지 시안은 화면 구조와 우선순위를 정하는 근거이며 실제 문구와 데이터 계약의 원장은 PAGE·UI·API 문서다.
- 모바일 화면은 브라우저 안에서 동작하는 웹앱을 기준으로 하며 iOS·Android 전용 상태 바나 제스처에 의존하지 않는다.
- 현재 이미지 시안은 페이지 내부 콘텐츠와 주요 CTA 검토에 집중하기 위해 전역 하단 내비게이션을 그리지 않는다. 실제 구현의 공통 내비게이션은 레이아웃 컴포넌트에서 별도로 검증한다.
- 공개 홈과 상품 상세의 개인화 영역이 실패해도 공개 콘텐츠를 계속 볼 수 있게 대체 상태를 둔다.
- 주문·결제·재고·쿠폰의 최종 판정은 서버 응답을 사용하고, 화면은 판정 결과와 최신 확인 시각을 명확히 보여준다.
- 기존 시안과 컴포넌트 시트는 삭제하지 않고 새 모바일 웹 시안의 근거로 함께 보관한다.

## 연관 태그

🏷️ 사이트맵 참조: [구매자 모바일 웹앱 사이트맵](../../10-sitemap/buyer-mobile-web/README.md) | 요구사항 참조: [REQ.A.08](../../00-requirements/REQ_A_08_web_application.md) | 웹 애플리케이션 참조: [WEB 설계](../../60-web-application/README.md)
