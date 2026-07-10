---
id: UI.A.203
group_id: UI.A.200
page_id: PAGE.A.203
title: 드롭 등록·편집 UI
type: ui-asset
status: draft
device: desktop-web
tags: [ui, seller, seller-portal, drop-editor, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# UI.A.203 드롭 등록·편집

## 기본 정보

- UI ID: `UI.A.203`
- 연관 Page: [PAGE.A.203 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 통합 원본: [UI.A.200~211 판매자 웹 포털](README.md)

## 화면 시안

![UI.A.203 드롭 등록·편집](../assets/UI_A_200_seller_portal/UI_A_203_01_drop_create_edit.png)

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
| 문서 상태 | `draft.dropId` | string? | 수정 시 대상 드롭 식별, 신규 등록 시 비어 있음 |
| 문서 상태 | `draft.status` | enum | 신규, 임시 저장, 반려 보완, 변경 요청 작성 상태 표시 |
| 문서 상태 | `draft.savedAt` | datetime? | 마지막 임시 저장 시각 표시 |
| 단계 표시 | `stepper.currentStep` | enum | 상품 정보, 판매 조건, 재고, 검수 요청 중 현재 단계 표시 |
| 단계 표시 | `stepper.completedSteps[]` | enum[] | 입력을 완료한 단계 표시 |
| 단계 표시 | `stepper.errorSteps[]` | enum[] | 검증 오류가 있는 단계 표시 |
| 상품 선택 | `form.productId` | string | 판매할 등록 상품 지정 |
| 상품 선택 | `form.productName` | string | 선택한 상품명 확인 |
| 판매 일정 | `form.displayStartsAt` | datetime | 구매자에게 드롭이 노출되기 시작하는 시각 입력 |
| 판매 일정 | `form.opensAt` | datetime | 구매가 시작되는 오픈 시각 입력 |
| 판매 일정 | `form.endsAt` | datetime | 구매가 종료되는 시각 입력 |
| 판매 조건 | `form.purchaseLimitPerUser` | number | 사용자별 최대 구매 수량 입력 |
| 판매 조건 | `form.shippingMethod` | enum | 택배, 직접 배송 등 배송 방식 선택 |
| 판매 조건 | `form.shippingNotice` | text | 배송 일정과 유의사항 입력 |
| 판매 조건 | `form.returnExchangeNotice` | text | 교환·반품 기준 입력 |
| 판매 조건 | `form.sellerNotice` | text | 구매자가 확인할 판매자 안내 입력 |
| 재고 배정 | `form.inventory[].optionId` | string | 판매 옵션 식별 |
| 재고 배정 | `form.inventory[].optionLabel` | string | 옵션명 표시 |
| 재고 배정 | `form.inventory[].totalQuantity` | number | 옵션별 전체 보유 재고 표시 |
| 재고 배정 | `form.inventory[].availableQuantity` | number | 배정 시점의 공급 가능 수량 표시 |
| 재고 배정 | `form.inventory[].reservedQuantity` | number | 다른 드롭에 이미 배정된 예약 재고 표시 |
| 재고 배정 | `form.inventory[].salesQuantity` | number | 드롭에 확정할 판매 수량 입력 |
| 재고 배정 | `form.inventory[].referenceAt` | datetime | 공급 가능 수량을 조회한 기준 시각 표시 |
| 재고 배정 | `form.inventory[].isSupplyAvailable` | boolean | 입력 수량의 공급 가능 여부 표시 |
| 재고 합계 | `form.inventoryTotalQuantity` | number | 모든 옵션의 판매 수량 합계 표시 |
| 미리보기 | `preview.thumbnailUrl` | image-url | 구매자 화면의 대표 이미지 미리보기 |
| 미리보기 | `preview.salePrice` | money | 구매자 화면의 판매가 미리보기 |
| 미리보기 | `preview.countdownTargetAt` | datetime | 카운트다운 기준 시각 미리보기 |
| 입력 검증 | `form.validationErrors[]` | field-error[] | 필드별 오류 코드와 안내 문구 표시 |
| 입력 검증 | `form.isDirty` | boolean | 저장되지 않은 변경 내용 존재 여부 표시 |
| 입력 검증 | `form.canSaveDraft` | boolean | 임시 저장 가능 여부 결정 |
| 입력 검증 | `form.canRequestReview` | boolean | 필수 입력 완료 후 검수 요청 가능 여부 결정 |

## 관련 문서

- [판매자 사이트맵](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- [판매자 요구사항](../../00-requirements/REQ_A_03_seller.md)
- [판매자 드롭 운영 유스케이스](../../30-uc/UC_A_02_seller_manage_drop.md)
