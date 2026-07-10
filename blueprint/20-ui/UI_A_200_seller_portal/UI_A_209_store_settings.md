---
id: UI.A.209
group_id: UI.A.200
page_id: PAGE.A.209
title: 판매자·스토어 정보 UI
type: ui-asset
status: draft
device: desktop-web
tags: [ui, seller, seller-portal, store-settings, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# UI.A.209 판매자·스토어 정보

## 기본 정보

- UI ID: `UI.A.209`
- 연관 Page: [PAGE.A.209 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 통합 원본: [UI.A.200~211 판매자 웹 포털](README.md)

## 화면 시안

![UI.A.209 판매자·스토어 정보](../assets/UI_A_200_seller_portal/UI_A_209_01_store_settings.png)

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
| 판매자 정보 | `form.sellerId` | string | 수정 대상 판매자 식별, 화면에서는 읽기 전용 표시 |
| 판매자 정보 | `form.sellerType` | enum | 계약된 판매자 유형 표시 |
| 판매자 정보 | `form.sellerStatus` | enum | 정상, 검증 대기, 이용 제한 상태 표시 |
| 담당자 정보 | `form.managerName` | string | 운영 담당자명 입력 |
| 담당자 정보 | `form.managerPhone` | phone | 운영 연락처 입력 |
| 담당자 정보 | `form.managerEmail` | email | 운영 이메일 입력 |
| 사업자 정보 | `form.businessRegistrationNumber` | string | 사업자등록번호 표시 및 검증 요청 |
| 사업자 정보 | `form.representativeName` | string | 대표자명 입력 |
| 사업자 정보 | `form.businessVerificationStatus` | enum | 미제출, 검증 중, 승인, 반려 상태 표시 |
| 사업자 정보 | `form.businessVerifiedAt` | datetime? | 사업자 검증 완료 시각 표시 |
| 스토어 브랜딩 | `form.storeDisplayName` | string | 구매자 화면의 스토어명 입력 |
| 스토어 브랜딩 | `form.logoUrl` | image-url? | 스토어 로고 업로드·미리보기 |
| 스토어 브랜딩 | `form.coverImageUrl` | image-url? | 스토어 상단 커버 이미지 업로드·미리보기 |
| 스토어 브랜딩 | `form.introduction` | text | 스토어 소개 문구 입력 |
| 구매자 안내 | `form.defaultShippingNotice` | text | 상품·드롭 등록 시 기본 배송 안내로 사용 |
| 구매자 안내 | `form.returnExchangeNotice` | text | 기본 교환·반품 안내로 사용 |
| 구매자 안내 | `form.customerContact` | string | 구매자에게 공개할 문의 경로 입력 |
| 스토어 미리보기 | `preview.storeDisplayName` | string | 저장 전 구매자 화면의 스토어명 확인 |
| 스토어 미리보기 | `preview.logoUrl` | image-url? | 저장 전 로고 표시 상태 확인 |
| 스토어 미리보기 | `preview.introduction` | text | 저장 전 소개 문구 표시 상태 확인 |
| 스토어 미리보기 | `preview.followerCount` | number | 구매자 화면에 표시되는 팔로워 수 확인 |
| 스토어 미리보기 | `preview.salesCount` | number | 구매자 화면에 표시되는 누적 판매 건수 확인 |
| 스토어 미리보기 | `preview.productCount` | number | 현재 스토어에 노출되는 상품 수 표시 |
| 스토어 미리보기 | `preview.averageRating` | number? | 구매자 화면에 제공되는 평균 평점 표시 |
| 스토어 미리보기 | `preview.reviewCount` | number | 평균 평점과 함께 표시되는 리뷰 수 확인 |
| 스토어 미리보기 | `preview.representativeProducts[]` | product-summary[] | 대표 상품의 이미지, 상품명, 가격, 평점 미리보기 |
| 스토어 미리보기 | `preview.sellerResponsibilityNotice` | text | 판매 책임 주체와 DropMong 역할 고지 확인 |
| 스토어 미리보기 | `preview.shippingNotice` | text | 구매자에게 공개되는 배송 안내 확인 |
| 스토어 미리보기 | `preview.returnExchangeNotice` | text | 구매자에게 공개되는 교환·반품 안내 확인 |
| 입력 검증 | `form.validationErrors[]` | field-error[] | 필드별 형식·필수값 오류 표시 |
| 입력 검증 | `form.isDirty` | boolean | 저장되지 않은 변경 내용 존재 여부 표시 |
| 입력 검증 | `form.canSave` | boolean | 권한과 검증 결과에 따른 저장 가능 여부 결정 |

## 관련 문서

- [판매자 사이트맵](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- [판매자 요구사항](../../00-requirements/REQ_A_03_seller.md)
- [판매자 드롭 운영 유스케이스](../../30-uc/UC_A_02_seller_manage_drop.md)
