---
id: sitemap-index
title: 사이트맵 인덱스
type: sitemap-index
status: draft
tags: [sitemap, flowchart, index, example]
source: local
created: 2026-07-06
updated: 2026-07-07
---

# 사이트맵 인덱스

## 역할

전체 페이지 구조와 페이지 간 연결을 flowchart로 보여주는 인덱스다.

## 템플릿

- [페이지 템플릿](.template/PAGE_A_XX.md)

## 주요 페이지

![userflow](assets/dropmong-user-flow.png)

## 전체 사이트맵

![sitemap](assets/dropmong-sitemap.png)

```mermaid
mindmap
  root((DropMong App))
    홈
      추천 드롭
      오픈 예정
      오늘의 큐레이션
      카테고리
      브랜드관
      실시간 랭킹
    드롭
      전체 드롭
      진행 예정
      진행 중
      종료된 드롭
      드롭 캘린더
    상품 상세
      상품 정보
      옵션 / 사이즈 선택
      카운트다운
      드롭 알림 받기
      구매 유의사항
      리뷰 / 문의
    검색 / 탐색
      통합 검색
      필터 / 정렬
      카테고리 탐색
      브랜드 검색
      최근 본 상품
    장바구니 · 주문 · 결제
      장바구니
      주문서
      배송지 선택
      쿠폰 / 포인트
      결제 수단 선택
      주문 완료
    알림
      드롭 오픈 알림
      관심 상품 알림
      주문 / 배송 알림
      마케팅 알림
      알림 설정
    마이
      프로필
      관심 상품
      주문 내역
      배송 조회
      쿠폰 / 포인트
      결제 수단 관리
      주소록
    고객지원 / 설정
      공지사항
      FAQ
      1:1 문의
      이용약관
      개인정보처리방침
      앱 설정
    인증
      로그인 메인
      이메일 로그인
      이메일 회원가입
      휴대폰 번호 로그인
      소셜 로그인
```

## 페이지 목록

### 실제 문서

- [PAGE.A.01 홈 화면](PAGE_A_01_homepage.md)
- [PAGE.A.02 상품 상세 페이지](PAGE_A_02_product_detail.md)
- [PAGE.A.06 장바구니 페이지](PAGE_A_06_shopping_cart.md)
- [PAGE.A.10 마이 페이지](PAGE_A_10_my.md)
- [PAGE.A.11 주문/결제 페이지](PAGE_A_11_payment.md)
- [PAGE.A.14 주문 완료 페이지](PAGE_A_14_order_complete.md)
- [PAGE.A.15 주문 내역 페이지](PAGE_A_15_order_history.md)
- [PAGE.A.16 배송 조회 페이지](PAGE_A_16_track_order.md)
- [PAGE.A.17 배송/주문 관리 페이지](PAGE_A_17_shipping_order_manage.md)
- [PAGE.A.30 로그인 메인 페이지](PAGE_A_30_multi_signin.md)
- [PAGE.A.31 이메일 회원가입 페이지](PAGE_A_31_email_signup.md)
- [PAGE.A.32 이메일 로그인 페이지](PAGE_A_32_email_signin.md)

### 예시 문서

- [PAGE.A.01 주문 결제](.examples/PAGE_A_01_order_checkout.md)
- [PAGE.A.02 상품 상세](.examples/PAGE_A_02_product_detail.md)
- [PAGE.A.03 구매자 프로필](.examples/PAGE_A_03_buyer_profile.md)

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.01](../00-requirements/.example/REQ_A_01_order_checkout.md) | 유스케이스 참조: [UC.A.01](../30-uc/.examples/UC_A_01_place_order.md) | 영속성 참조: [PST.A.01](../55-persistence/.examples/PST_A_01_order_persistence.md) | 서비스 참조: [SVC.A.01](../60-service/.examples/SVC_A_01_order_service.md) | 시나리오 참조: [SCN.A.01](../80-scenario/.examples/SCN_A_01_place_order.md)
