---
id: UI.A.207
group_id: UI.A.200
page_id: PAGE.A.207
title: 판매 분석 UI
type: ui-asset
status: draft
device: desktop-web
tags: [ui, seller, seller-portal, sales-analytics, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# UI.A.207 판매 분석

## 기본 정보

- UI ID: `UI.A.207`
- 연관 Page: [PAGE.A.207 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 통합 원본: [UI.A.200~211 판매자 웹 포털](README.md)

## 화면 시안

![UI.A.207 판매 분석](../assets/UI_A_200_seller_portal/UI_A_207_01_sales_analytics.png)

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
| 조회 조건 | `filters.dateRange` | date-range | 분석 대상 기간 선택 |
| 조회 조건 | `filters.productId` | string? | 특정 상품 기준으로 분석 범위 제한 |
| 조회 조건 | `filters.optionId` | string? | 특정 상품 옵션 기준으로 분석 범위 제한 |
| 조회 조건 | `filters.dropId` | string? | 특정 드롭 기준으로 분석 범위 제한 |
| 조회 조건 | `filters.dropRound` | number? | 같은 상품의 특정 드롭 회차 선택 |
| 조회 조건 | `filters.couponId` | string? | 특정 쿠폰의 효과 분석 |
| 조회 조건 | `filters.acquisitionChannel` | enum? | 검색, 홈, 알림, 외부 링크 등 유입 경로 선택 |
| 조회 조건 | `filters.couponUsage` | enum? | 전체, 쿠폰 사용, 쿠폰 미사용 주문 구분 |
| 조회 조건 | `filters.cancellationRefundStatus` | enum? | 전체, 정상, 취소, 환불 주문 구분 |
| 데이터 기준 | `analytics.aggregationCompletedAt` | datetime | 마지막 집계 완료 시각 표시 |
| 데이터 기준 | `analytics.isRealtimeEstimate` | boolean | 실시간 추정치 포함 여부 표시 |
| 데이터 기준 | `analytics.comparisonRange` | date-range? | 이전 기간 비교 기준 표시 |
| 핵심 지표 | `analytics.viewCount` | number | 상품·드롭 노출 및 조회 수 표시 |
| 핵심 지표 | `analytics.notificationCount` | number | 오픈 알림 설정 수 표시 |
| 핵심 지표 | `analytics.purchaseAttemptCount` | number | 구매 또는 결제 시도 수 표시 |
| 핵심 지표 | `analytics.inventoryAllocationCount` | number | 구매 시도 중 재고 배정이 완료된 수 표시 |
| 핵심 지표 | `analytics.orderSuccessCount` | number | 결제 완료 주문 수 표시 |
| 핵심 지표 | `analytics.paymentFailureCount` | number | 결제 실패 건수 표시 |
| 핵심 지표 | `analytics.conversionRate` | percentage | 조회 또는 구매 시도 대비 주문 성공률 표시 |
| 핵심 지표 | `analytics.salesAmount` | money | 결제 완료 기준 매출액 표시 |
| 핵심 지표 | `analytics.soldOutAt` | datetime? | 품절된 경우 품절 시각 표시 |
| 핵심 지표 | `analytics.comparedWithPreviousPeriodRates` | percentage-map | 조회·알림·구매 시도·구매 성공·전환율의 이전 비교 기간 대비 증감률 표시 |
| 구매 단계 | `analytics.funnel[].stage` | enum | 조회, 알림 설정, 구매 시도, 주문 성공 단계 구분 |
| 구매 단계 | `analytics.funnel[].count` | number | 단계별 사용자 또는 이벤트 수 표시 |
| 구매 단계 | `analytics.funnel[].conversionRate` | percentage | 이전 단계 대비 전환율 표시 |
| 추이 차트 | `analytics.trend[].date` | date | 차트 기준 날짜 표시 |
| 추이 차트 | `analytics.trend[].salesAmount` | money | 날짜별 매출액 표시 |
| 추이 차트 | `analytics.trend[].orderCount` | number | 날짜별 결제 완료 주문 수 표시 |
| 상품 성과 | `analytics.products[].productId` | string | 상품 관리 페이지 연결 식별자 |
| 상품 성과 | `analytics.products[].productName` | string | 상품명 표시 |
| 상품 성과 | `analytics.products[].viewCount` | number | 상품별 조회 수 표시 |
| 상품 성과 | `analytics.products[].orderSuccessCount` | number | 상품별 주문 성공 수 표시 |
| 상품 성과 | `analytics.products[].conversionRate` | percentage | 상품별 전환율 표시 |
| 상품 성과 | `analytics.products[].salesAmount` | money | 상품별 매출액 표시 |
| 쿠폰 효과 | `analytics.couponEffect.usedOrderSales` | money | 쿠폰 사용 주문의 매출액 표시 |
| 쿠폰 효과 | `analytics.couponEffect.useCount` | number | 분석 기간의 쿠폰 사용 수 표시 |
| 쿠폰 효과 | `analytics.couponEffect.usedOrderRate` | percentage | 전체 주문 중 쿠폰 사용 주문 비율 표시 |
| 쿠폰 효과 | `analytics.couponEffect.sellerCostAmount` | money | 판매자 부담 쿠폰 비용 표시 |
| 쿠폰 효과 | `analytics.couponEffect.conversionLift` | percentage | 미사용 주문군 대비 전환율 차이 표시 |
| 쿠폰 효과 | `analytics.couponEffect.roi` | number | 판매자 부담 비용 대비 추가 매출 배수 표시 |
| 취소·환불 | `analytics.cancellationRefundAmount` | money | 취소·환불된 총 금액 표시 |
| 취소·환불 | `analytics.cancellationRefundRequestCount` | number | 취소·환불 요청 건수 표시 |
| 취소·환불 | `analytics.cancellationReasons[]` | reason-count[] | 사유별 취소·환불 건수 표시 |
| 실패 분석 | `analytics.failureReasons[]` | reason-count[] | 구매·결제 실패 사유별 건수와 비율 표시 |

## 관련 문서

- [판매자 사이트맵](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- [판매자 요구사항](../../00-requirements/REQ_A_03_seller.md)
- [판매자 드롭 운영 유스케이스](../../30-uc/UC_A_02_seller_manage_drop.md)
