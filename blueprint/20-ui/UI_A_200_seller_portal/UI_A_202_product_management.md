---
id: UI.A.202
group_id: UI.A.200
page_id: PAGE.A.202
title: 상품 관리 UI
type: ui-asset
status: draft
device: desktop-web
tags: [ui, seller, seller-portal, product-management, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# UI.A.202 상품 관리

## 기본 정보

- UI ID: `UI.A.202`
- 연관 Page: [PAGE.A.202 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 통합 원본: [UI.A.200~211 판매자 웹 포털](README.md)

## 화면 시안

![UI.A.202 상품 관리](../assets/UI_A_200_seller_portal/UI_A_202_01_product_management.png)

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
| 검색 조건 | `filters.keyword` | string | 상품명, 상품 코드 또는 옵션명 검색 |
| 검색 조건 | `filters.categoryId` | string? | 판매 카테고리 필터 |
| 검색 조건 | `filters.saleStatus` | enum[] | 판매 가능, 품절, 판매 중지 상태 필터 |
| 검색 조건 | `filters.sort` | enum | 최근 수정일, 재고 적음, 판매량 순 정렬 |
| 상태 요약 | `summary.totalCount` | number | 등록된 전체 상품 수 표시 |
| 상태 요약 | `summary.saleableCount` | number | 현재 판매 가능한 상품 수 표시 |
| 상태 요약 | `summary.lowStockCount` | number | 재고 부족 기준 이하 상품 수 표시 |
| 상태 요약 | `summary.comparedWithPreviousPeriodRates` | percentage-map | 전체·판매 가능·재고 부족 건수의 이전 비교 기간 대비 증감률 표시 |
| 상품 목록 | `products[].productId` | string | 상품 상세·수정 페이지 연결 식별자 |
| 상품 목록 | `products[].sellerSku` | string | 판매자가 관리하는 상품 코드 표시 |
| 상품 목록 | `products[].productName` | string | 상품명 표시 |
| 상품 목록 | `products[].thumbnailUrl` | image-url | 대표 이미지 표시 |
| 상품 목록 | `products[].categoryName` | string | 등록 카테고리 표시 |
| 상품 목록 | `products[].optionSummary` | string | 색상·사이즈 등 옵션 구성 요약 표시 |
| 상품 목록 | `products[].salePrice` | money | 현재 판매가 표시 |
| 상품 목록 | `products[].stock.totalQuantity` | number | 전체 보유 재고 표시 |
| 상품 목록 | `products[].stock.availableQuantity` | number | 드롭에 배정 가능한 재고 표시 |
| 상품 목록 | `products[].stock.status` | enum | 정상, 재고 부족, 품절 상태 표시 |
| 상품 목록 | `products[].linkedDropCount` | number | 상품이 연결된 드롭 수 표시 |
| 상품 목록 | `products[].saleStatus` | enum | 판매 가능, 품절, 판매 중지 상태 표시 |
| 상품 목록 | `products[].updatedAt` | datetime | 마지막 수정 시각 표시 |
| 작업 상태 | `products[].actions.canPreview` | boolean | 구매자 노출 미리보기 제공 여부 결정 |
| 작업 상태 | `products[].actions.canEdit` | boolean | 현재 페이지의 상품 편집 화면 진입 가능 여부 결정 |
| 상품 편집 | `editor.productId` | string? | 수정 대상 상품 식별, 신규 등록 시 비어 있음 |
| 상품 편집 | `editor.productName` | string | 구매자에게 표시할 상품명 입력 |
| 상품 편집 | `editor.categoryId` | string | 상품 카테고리 선택 |
| 상품 편집 | `editor.thumbnailUrl` | image-url | 목록과 드롭 카드에 사용할 대표 이미지 입력 |
| 상품 편집 | `editor.detailImageUrls[]` | image-url[] | 상품 상세 본문에 노출할 이미지 입력·정렬 |
| 상품 편집 | `editor.description` | rich-text | 상품 특징, 소재, 규격 등 상세 설명 입력 |
| 상품 편집 | `editor.salePrice` | money | 기본 판매가 입력 |
| 상품 편집 | `editor.options[].optionId` | string? | 기존 옵션 식별, 신규 옵션은 저장 시 발급 |
| 상품 편집 | `editor.options[].sellerSku` | string | 옵션별 판매자 상품 코드 입력 |
| 상품 편집 | `editor.options[].optionLabel` | string | 색상·사이즈 조합명 입력 |
| 상품 편집 | `editor.options[].stockQuantity` | number | 옵션별 보유 재고 입력 |
| 구매자 미리보기 | `preview.sellerDisplayName` | string | 상품 상세에 노출될 판매자명 확인 |
| 구매자 미리보기 | `preview.sellerResponsibilityNotice` | text | 판매 책임 주체와 DropMong 역할 고지 확인 |
| 구매자 미리보기 | `preview.shippingNotice` | text | 상품 상세·주문 화면의 배송 안내 확인 |
| 구매자 미리보기 | `preview.returnExchangeNotice` | text | 상품 상세·주문 화면의 교환·반품 안내 확인 |
| 일괄 선택 | `selectedProductIds[]` | string[] | 일괄 상태 변경 대상 상품 식별 |
| 페이지네이션 | `pagination.page`, `pageSize`, `totalCount` | number | 목록 페이지와 전체 결과 수 표시 |

## 관련 문서

- [판매자 사이트맵](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- [판매자 요구사항](../../00-requirements/REQ_A_03_seller.md)
- [판매자 드롭 운영 유스케이스](../../30-uc/UC_A_02_seller_manage_drop.md)
