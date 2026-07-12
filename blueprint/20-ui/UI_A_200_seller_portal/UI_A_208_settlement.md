---
id: UI.A.208
group_id: UI.A.200
page_id: PAGE.A.208
title: 정산 조회 UI
type: ui-asset
status: draft
device: desktop-web
tags: [ui, seller, seller-portal, settlement, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# UI.A.208 정산 조회

## 기본 정보

- UI ID: `UI.A.208`
- 연관 Page: [PAGE.A.208 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 통합 원본: [UI.A.200~211 판매자 웹 포털](README.md)

## 화면 시안

![UI.A.208 정산 조회](../assets/UI_A_200_seller_portal/UI_A_208_01_settlement.png)

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
| 조회 조건 | `filters.settlementPeriod` | date-range | 정산 대상 기간 필터 |
| 조회 조건 | `filters.dropId` | string? | 특정 드롭의 정산만 조회 |
| 조회 조건 | `filters.status` | enum[] | 예정, 보류, 확정, 지급 완료 상태 필터 |
| 상태 요약 | `summary.expectedAmount` | money | 지급 예정 정산액 합계 표시 |
| 상태 요약 | `summary.onHoldAmount` | money | 보류 중인 정산액 합계 표시 |
| 상태 요약 | `summary.confirmedAmount` | money | 확정된 정산액 합계 표시 |
| 상태 요약 | `summary.deductionAmount` | money | 조회 기간의 차감액 합계 표시 |
| 상태 요약 | `summary.comparedWithPreviousPeriodRates` | percentage-map | 예정·보류·확정·차감 금액의 이전 비교 기간 대비 증감률 표시 |
| 정산 목록 | `settlements[].settlementId` | string | 정산 상세 식별자 |
| 정산 목록 | `settlements[].periodStart` | date | 정산 대상 시작일 표시 |
| 정산 목록 | `settlements[].periodEnd` | date | 정산 대상 종료일 표시 |
| 정산 목록 | `settlements[].targetOrderCount` | number | 정산에 포함된 결제 완료 주문 수 표시 |
| 정산 금액 | `settlements[].grossSalesAmount` | money | 취소·환불 전 주문 매출 합계 표시 |
| 정산 금액 | `settlements[].sellerCouponCostAmount` | money | 판매자 부담 쿠폰 비용 표시 |
| 정산 금액 | `settlements[].platformFeeAmount` | money | 플랫폼 이용 수수료 표시 |
| 정산 금액 | `settlements[].deductionAmount` | money | 환불·분쟁 등 차감 금액 표시 |
| 정산 금액 | `settlements[].expectedPayoutAmount` | money | 최종 지급 예정 금액 표시 |
| 정산 상태 | `settlements[].status` | enum | 예정, 보류, 확정, 지급 완료 상태 표시 |
| 정산 상태 | `settlements[].expectedPayoutAt` | date? | 지급 예정일 표시 |
| 정산 상태 | `settlements[].confirmedAt` | datetime? | 정산 확정 시각 표시 |
| 보류·차감 | `settlements[].holdReasonCode` | enum? | 정산 보류 사유 식별 |
| 보류·차감 | `settlements[].holdReasonLabel` | string? | 판매자가 확인할 보류 사유 표시 |
| 보류·차감 | `settlements[].deductionReason` | text? | 차감 근거와 설명 표시 |
| 보류·차감 | `settlements[].relatedOrderIds[]` | string[] | 보류·차감과 관련된 주문 연결 |
| 페이지네이션 | `pagination.page`, `pageSize`, `totalCount` | number | 목록 페이지와 전체 결과 수 표시 |

## 관련 문서

- [판매자 사이트맵](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- [판매자 요구사항](../../00-requirements/REQ_A_03_seller.md)
- [판매자 드롭 운영 유스케이스](../../30-uc/UC_A_02_seller_manage_drop.md)
