---
id: UI.A.201
group_id: UI.A.200
page_id: PAGE.A.201
title: 드롭 관리 UI
type: ui-asset
status: draft
device: desktop-web
tags: [ui, seller, seller-portal, drop-management, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# UI.A.201 드롭 관리

## 기본 정보

- UI ID: `UI.A.201`
- 연관 Page: [PAGE.A.201 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 통합 원본: [UI.A.200~211 판매자 웹 포털](README.md)

## 화면 시안

![UI.A.201 드롭 관리](../assets/UI_A_200_seller_portal/UI_A_201_01_drop_management.png)

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
| 검색 조건 | `filters.keyword` | string | 드롭명 또는 상품명 검색 |
| 검색 조건 | `filters.status` | enum[] | 임시 저장, 검수 중, 승인, 반려, 예정, 진행 중, 종료 필터 |
| 검색 조건 | `filters.scheduleRange` | date-range | 오픈 또는 종료 시각 범위 필터 |
| 검색 조건 | `filters.sort` | enum | 최근 수정일, 오픈 임박, 판매량 순 정렬 |
| 상태 요약 | `summary.totalCount` | number | 조회 조건에 해당하는 전체 드롭 수 표시 |
| 상태 요약 | `summary.activeCount` | number | 진행 중인 드롭 수 표시 |
| 상태 요약 | `summary.reviewRequiredCount` | number | 반려 또는 보완이 필요한 드롭 수 표시 |
| 상태 요약 | `summary.comparedWithPreviousPeriodRates` | percentage-map | 전체·진행 중·검수 필요 건수의 이전 비교 기간 대비 증감률 표시 |
| 드롭 목록 | `drops[].dropId` | string | 상세·수정·검수 요청 페이지 연결 식별자 |
| 드롭 목록 | `drops[].dropName` | string | 판매자가 설정한 드롭명 표시 |
| 드롭 목록 | `drops[].productName` | string | 대표 상품명 표시 |
| 드롭 목록 | `drops[].thumbnailUrl` | image-url | 대표 상품 이미지 표시 |
| 드롭 목록 | `drops[].status` | enum | 현재 드롭 운영 상태 표시 |
| 드롭 목록 | `drops[].reviewStatus` | enum | 검수 전, 검수 중, 승인, 반려 상태 표시 |
| 드롭 목록 | `drops[].opensAt` | datetime | 오픈 예정 또는 시작 시각 표시 |
| 드롭 목록 | `drops[].endsAt` | datetime | 판매 종료 시각 표시 |
| 드롭 목록 | `drops[].inventory.totalQuantity` | number | 확정 판매 수량 표시 |
| 드롭 목록 | `drops[].inventory.remainingQuantity` | number | 남은 판매 가능 수량 표시 |
| 드롭 목록 | `drops[].inventory.soldRate` | percentage | 판매 소진율 표시 |
| 드롭 목록 | `drops[].orderSuccessCount` | number | 결제 완료 주문 건수 표시 |
| 반려 정보 | `drops[].rejectionReasonSummary` | string? | 반려된 드롭의 대표 보완 사유 표시 |
| 작업 상태 | `drops[].actions.canEdit` | boolean | 수정 페이지 진입 가능 여부 결정 |
| 작업 상태 | `drops[].actions.disabledReason` | string? | 수정·삭제가 제한된 이유 표시 |
| 페이지네이션 | `pagination.page`, `pageSize`, `totalCount` | number | 목록 페이지와 전체 결과 수 표시 |

## 관련 문서

- [판매자 사이트맵](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- [판매자 요구사항](../../00-requirements/REQ_A_03_seller.md)
- [판매자 드롭 운영 유스케이스](../../30-uc/UC_A_02_seller_manage_drop.md)
