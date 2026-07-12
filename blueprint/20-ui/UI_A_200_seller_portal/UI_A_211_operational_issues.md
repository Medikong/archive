---
id: UI.A.211
group_id: UI.A.200
page_id: PAGE.A.211
title: 운영 이슈 UI
type: ui-asset
status: draft
device: desktop-web
tags: [ui, seller, seller-portal, operational-issues, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# UI.A.211 운영 이슈

## 기본 정보

- UI ID: `UI.A.211`
- 연관 Page: [PAGE.A.211 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 통합 원본: [UI.A.200~211 판매자 웹 포털](README.md)

## 화면 시안

![UI.A.211 운영 이슈](../assets/UI_A_200_seller_portal/UI_A_211_01_operational_issues.png)

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
| 상태 요약 | `summary.waitingCount` | number | 판매자 확인 또는 추가 정보가 필요한 이슈 수 표시 |
| 상태 요약 | `summary.inProgressCount` | number | 처리 중인 이슈 수 표시 |
| 상태 요약 | `summary.resolvedCount` | number | 조회 기간에 해결된 이슈 수 표시 |
| 검색 조건 | `filters.keyword` | string | 이슈번호, 주문번호 또는 상품명 검색 |
| 검색 조건 | `filters.issueType` | enum[] | 출고 지연, 재고 불일치, 취소·환불, 정산 문의 구분 |
| 검색 조건 | `filters.status` | enum[] | 접수, 확인 대기, 처리 중, 해결 상태 필터 |
| 검색 조건 | `filters.dropId` | string? | 특정 드롭과 관련된 이슈만 조회 |
| 이슈 목록 | `issues[].issueId` | string | 이슈 상세 식별자 |
| 이슈 목록 | `issues[].issueType` | enum | 이슈 유형 표시 |
| 이슈 목록 | `issues[].reasonCode` | enum | 표준 원인 코드 표시 |
| 이슈 목록 | `issues[].status` | enum | 접수, 확인 대기, 처리 중, 해결 상태 표시 |
| 이슈 목록 | `issues[].orderId` | string? | 관련 주문 상세 페이지 연결 |
| 이슈 목록 | `issues[].orderNumber` | string? | 관련 판매자용 주문번호 표시 |
| 이슈 목록 | `issues[].dropId` | string? | 관련 드롭 관리 페이지 연결 |
| 이슈 목록 | `issues[].productName` | string? | 관련 상품명 표시 |
| 이슈 목록 | `issues[].receivedAt` | datetime | 이슈 접수 시각 표시 |
| 이슈 목록 | `issues[].resolvedAt` | datetime? | 해결된 경우 처리 완료 시각 표시 |
| 이슈 상세 | `issueDetail.description` | text | 접수된 문제 설명 표시 |
| 이슈 상세 | `issueDetail.sellerMemo` | text? | 판매자 내부 확인 메모 입력·표시 |
| 이슈 상세 | `issueDetail.assigneeDisplayName` | string? | 처리 담당자가 배정된 경우 담당자명 표시 |
| 이슈 상세 | `issueDetail.attachments[]` | file[] | 증빙 이미지·문서 표시 |
| 처리 이력 | `issueDetail.timeline[].status` | enum | 접수부터 해결까지의 처리 상태 표시 |
| 처리 이력 | `issueDetail.timeline[].occurredAt` | datetime | 상태 변경 시각 표시 |
| 처리 이력 | `issueDetail.timeline[].note` | text? | 담당자 또는 판매자의 처리 메모 표시 |
| 이슈 접수 | `report.orderId` | string? | 새 이슈와 관련된 주문 지정 |
| 이슈 접수 | `report.issueType` | enum | 새 이슈 유형 선택 |
| 이슈 접수 | `report.reasonCode` | enum | 새 이슈 원인 선택 |
| 이슈 접수 | `report.description` | text | 판매자가 확인한 문제 내용 입력 |
| 이슈 접수 | `report.attachmentUrls[]` | file-url[] | 이슈 증빙 파일 첨부 |
| 페이지네이션 | `pagination.page`, `pageSize`, `totalCount` | number | 목록 페이지와 전체 결과 수 표시 |

## 관련 문서

- [판매자 사이트맵](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- [판매자 요구사항](../../00-requirements/REQ_A_03_seller.md)
- [판매자 드롭 운영 유스케이스](../../30-uc/UC_A_02_seller_manage_drop.md)
