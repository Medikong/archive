---
id: UI.A.205
group_id: UI.A.200
page_id: PAGE.A.205
title: 주문·출고 UI
type: ui-asset
status: draft
device: desktop-web
tags: [ui, seller, seller-portal, order-fulfillment, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# UI.A.205 주문·출고

## 기본 정보

- UI ID: `UI.A.205`
- 연관 Page: [PAGE.A.205 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 통합 원본: [UI.A.200~211 판매자 웹 포털](README.md)

## 화면 시안

![UI.A.205 주문·출고](../assets/UI_A_200_seller_portal/UI_A_205_01_order_fulfillment.png)

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
| 상태 요약 | `summary.shippingPendingCount` | number | 송장 등록 또는 출고 처리가 필요한 주문 수 표시 |
| 상태 요약 | `summary.dueTodayCount` | number | 오늘 출고 기한이 도래하는 주문 수 표시 |
| 상태 요약 | `summary.inTransitCount` | number | 배송 중인 주문 수 표시 |
| 상태 요약 | `summary.delayedCount` | number | 출고·배송 지연 주문 수 표시 |
| 상태 요약 | `summary.comparedWithPreviousPeriodRates` | percentage-map | 출고 대기·오늘 마감·배송 중·지연 건수의 이전 비교 기간 대비 증감률 표시 |
| 검색 조건 | `filters.keyword` | string | 주문번호, 상품명 또는 구매자 마스킹 정보 검색 |
| 검색 조건 | `filters.fulfillmentStatus` | enum[] | 출고 대기, 배송 중, 배송 완료, 지연 상태 필터 |
| 검색 조건 | `filters.orderedAtRange` | date-range | 주문 접수 기간 필터 |
| 검색 조건 | `filters.dropId` | string? | 특정 드롭의 주문만 조회 |
| 주문 목록 | `orders[].orderId` | string | 주문 상세 및 운영 이슈 연결 식별자 |
| 주문 목록 | `orders[].orderNumber` | string | 판매자용 주문번호 표시 |
| 주문 목록 | `orders[].orderedAt` | datetime | 주문 접수 시각 표시 |
| 주문 목록 | `orders[].productName` | string | 주문 상품명 표시 |
| 주문 목록 | `orders[].thumbnailUrl` | image-url | 주문 상품 대표 이미지 표시 |
| 주문 목록 | `orders[].optionLabel` | string | 색상·사이즈 등 주문 옵션 표시 |
| 주문 목록 | `orders[].quantity` | number | 주문 수량 표시 |
| 주문 목록 | `orders[].buyer.maskedName` | string | 최소 범위로 마스킹한 구매자명 표시 |
| 주문 목록 | `orders[].buyer.maskedPhone` | string? | 출고에 필요한 경우에만 마스킹 연락처 표시 |
| 주문 목록 | `orders[].paymentAmount` | money | 결제 완료 금액 표시 |
| 출고 정보 | `orders[].fulfillmentStatus` | enum | 출고 대기, 출고 완료, 배송 중, 배송 완료, 지연 상태 표시 |
| 출고 정보 | `orders[].carrierCode` | enum? | 선택한 택배사 코드 표시 |
| 출고 정보 | `orders[].carrierName` | string? | 택배사명 표시 |
| 출고 정보 | `orders[].trackingNumber` | string? | 등록된 송장번호 표시 |
| 출고 정보 | `orders[].shippingDeadlineAt` | datetime? | 약속된 출고 처리 기한 표시 |
| 출고 정보 | `orders[].shippedAt` | datetime? | 실제 출고 처리 시각 표시 |
| 일괄 선택 | `selectedOrderIds[]` | string[] | 송장 일괄 등록·자료 다운로드 대상 식별 |
| 다운로드 요청 | `download.purpose` | enum | 배송 처리, 고객 문의, 정산 확인 등 다운로드 목적 기록 |
| 다운로드 요청 | `download.range` | enum/date-range | 내려받을 주문 범위 확인 |
| 다운로드 요청 | `download.format` | enum | XLSX, CSV, PDF 파일 형식 선택 |
| 다운로드 요청 | `download.requestedByDisplayName` | string | 다운로드 요청자 확인 |
| 다운로드 요청 | `download.requestedAt` | datetime? | 다운로드 요청 시각 표시, 요청 전에는 비어 있음 |
| 다운로드 요청 | `download.accessLogNotice` | string | 다운로드 목적과 접근 기록이 보관됨을 안내 |
| 다운로드 요청 | `download.expiresAt` | datetime? | 생성된 파일의 만료 시각 표시 |
| 페이지네이션 | `pagination.page`, `pageSize`, `totalCount` | number | 목록 페이지와 전체 결과 수 표시 |

## 관련 문서

- [판매자 사이트맵](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- [판매자 요구사항](../../00-requirements/REQ_A_03_seller.md)
- [판매자 드롭 운영 유스케이스](../../30-uc/UC_A_02_seller_manage_drop.md)
