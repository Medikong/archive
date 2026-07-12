---
id: UI.A.206
group_id: UI.A.200
page_id: PAGE.A.206
title: 쿠폰·프로모션 UI
type: ui-asset
status: draft
device: desktop-web
tags: [ui, seller, seller-portal, coupon-promotion, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# UI.A.206 쿠폰·프로모션

## 기본 정보

- UI ID: `UI.A.206`
- 연관 Page: [PAGE.A.206 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 통합 원본: [UI.A.200~211 판매자 웹 포털](README.md)

## 화면 시안

![UI.A.206 쿠폰·프로모션](../assets/UI_A_200_seller_portal/UI_A_206_01_coupon_promotion.png)

## 공통 화면 필드

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 판매자 식별 | `seller.id` | string | 모든 조회·수정 요청의 판매자 범위를 고정 |
| 판매자 식별 | `seller.displayName` | string | 상단 바와 계정 전환 영역에 현재 판매자명 표시 |
| 판매자 식별 | `seller.type` | enum | 셀러·브랜드·제휴 판매자 등 계약 유형 표시 |
| 판매자 상태 | `seller.status` | enum | 정상, 검증 대기, 이용 제한, 탈퇴 상태 표시 |
| 판매자 상태 | `seller.verificationStatus` | enum | 사업자 및 판매자 인증 완료 여부 표시 |
| 사용자 식별 | `member.id` | string | 현재 로그인한 판매자 구성원 식별 |
| 사용자 식별 | `member.displayName` | string | 상단 프로필과 감사 기록에 사용자명 표시 |
| 사용자 권한 | `member.role` | enum | 대표 관리자, 상품 담당자, 출고 담당자, 성과 조회자 구분 |
| 사용자 권한 | `member.permissions[]` | enum[] | 페이지와 입력 항목의 조회·수정 가능 여부 결정 |
| 전역 알림 | `navigation.unreadNotificationCount` | number | 상단 알림 아이콘에 읽지 않은 알림 수 표시 |
| 전역 알림 | `navigation.notifications[].targetPageId` | page-id | 알림 선택 시 이동할 실제 판매자 페이지 지정 |

## 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 상태 요약 | `summary.activeCouponCount` | number | 현재 발급 또는 사용 가능한 쿠폰 수 표시 |
| 상태 요약 | `summary.issuedCount` | number | 조회 기간의 쿠폰 발급 수 표시 |
| 상태 요약 | `summary.usedCount` | number | 결제에 사용된 쿠폰 수 표시 |
| 상태 요약 | `summary.sellerDiscountCost` | money | 판매자가 부담한 할인 금액 표시 |
| 상태 요약 | `summary.comparedWithPreviousPeriodRates` | percentage-map | 활성·발급·사용·할인 비용의 이전 비교 기간 대비 증감률 표시 |
| 검색 조건 | `filters.keyword` | string | 쿠폰명 또는 프로모션명 검색 |
| 검색 조건 | `filters.status` | enum[] | 예정, 진행 중, 종료, 중지 상태 필터 |
| 검색 조건 | `filters.period` | date-range | 발급 또는 사용 기간 필터 |
| 쿠폰 목록 | `coupons[].couponId` | string | 쿠폰 상세·수정 식별자 |
| 쿠폰 목록 | `coupons[].title` | string | 판매자와 구매자에게 표시할 쿠폰명 |
| 쿠폰 혜택 | `coupons[].benefitType` | enum | 정액, 정률, 무료 배송 혜택 구분 |
| 쿠폰 혜택 | `coupons[].benefitValue` | money/percentage | 할인 금액 또는 할인율 표시 |
| 쿠폰 혜택 | `coupons[].maxDiscountAmount` | money? | 정률 쿠폰의 최대 할인 금액 표시 |
| 쿠폰 혜택 | `coupons[].minimumOrderAmount` | money? | 쿠폰 사용을 위한 최소 주문 금액 표시 |
| 적용 대상 | `coupons[].targetScope` | enum | 전체 상품, 선택 상품, 선택 드롭 적용 범위 표시 |
| 적용 대상 | `coupons[].targetProductIds[]` | string[] | 쿠폰 적용 상품 식별 |
| 적용 대상 | `coupons[].targetDropIds[]` | string[] | 쿠폰 적용 드롭 식별 |
| 운영 기간 | `coupons[].issueStartsAt` | datetime | 쿠폰 발급 시작 시각 표시 |
| 운영 기간 | `coupons[].issueEndsAt` | datetime | 쿠폰 발급 종료 시각 표시 |
| 운영 기간 | `coupons[].useStartsAt` | datetime | 쿠폰 사용 시작 시각 표시 |
| 운영 기간 | `coupons[].useEndsAt` | datetime | 쿠폰 사용 종료 시각 표시 |
| 발급 제한 | `coupons[].issueQuantity` | number? | 전체 발급 한도 표시, 무제한이면 비어 있음 |
| 발급 제한 | `coupons[].perUserLimit` | number | 사용자별 발급 가능 수량 표시 |
| 운영 상태 | `coupons[].status` | enum | 예정, 진행 중, 종료, 중지 상태 표시 |
| 운영 상태 | `coupons[].approvalStatus` | enum? | 승인 대상 프로모션의 검토 상태 표시 |
| 성과 | `coupons[].issuedCount` | number | 쿠폰별 발급 수 표시 |
| 성과 | `coupons[].usedCount` | number | 쿠폰별 사용 수 표시 |
| 성과 | `coupons[].canceledOrReclaimedCount` | number | 취소 또는 회수된 쿠폰 수 표시 |
| 성과 | `coupons[].usageRate` | percentage | 발급 대비 사용 비율 표시 |
| 비용 | `coupons[].sellerCostAmount` | money | 판매자 부담 할인 금액 표시 |
| 비용 | `coupons[].platformCostAmount` | money | 플랫폼 부담 할인 금액 표시 |
| 제휴 제안 | `partnerships[].proposalId` | string | 플랫폼 제휴 프로모션 제안 식별 |
| 제휴 제안 | `partnerships[].title` | string | 제휴 프로모션명 표시 |
| 제휴 제안 | `partnerships[].benefitSummary` | string | 할인율·최대 할인 금액 등 제안 혜택 표시 |
| 제휴 제안 | `partnerships[].targetSummary` | string | 제안이 적용되는 상품·카테고리 범위 표시 |
| 제휴 제안 | `partnerships[].promotionPeriod` | date-range | 제휴 프로모션 예정 기간 표시 |
| 제휴 제안 | `partnerships[].linkedDropIds[]` | string[] | 제휴 쿠폰이 연결될 판매자 드롭 식별 |
| 제휴 제안 | `partnerships[].responseDeadlineAt` | datetime | 수락·거절 응답 기한 표시 |
| 제휴 제안 | `partnerships[].sellerCostRate` | percentage | 판매자 부담 비율 표시 |
| 제휴 제안 | `partnerships[].platformCostRate` | percentage | 플랫폼 부담 비율 표시 |
| 제휴 제안 | `partnerships[].responseStatus` | enum | 응답 대기, 수락, 거절, 만료 상태 표시 |

## 관련 문서

- [판매자 사이트맵](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- [판매자 요구사항](../../00-requirements/REQ_A_03_seller.md)
- [판매자 드롭 운영 유스케이스](../../30-uc/UC_A_02_seller_manage_drop.md)
