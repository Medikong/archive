---
id: UI.A.210
group_id: UI.A.200
page_id: PAGE.A.210
title: 팀·권한 UI
type: ui-asset
status: draft
device: desktop-web
tags: [ui, seller, seller-portal, team-permissions, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# UI.A.210 팀·권한

## 기본 정보

- UI ID: `UI.A.210`
- 연관 Page: [PAGE.A.210 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 통합 원본: [UI.A.200~211 판매자 웹 포털](README.md)

## 화면 시안

![UI.A.210 팀·권한](../assets/UI_A_200_seller_portal/UI_A_210_01_team_permissions.png)

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
| 구성원 목록 | `members[].memberId` | string | 구성원 식별 및 권한 변경 대상 지정 |
| 구성원 목록 | `members[].displayName` | string | 구성원명 표시 |
| 구성원 목록 | `members[].maskedEmail` | string | 초대·로그인 계정을 마스킹해 표시 |
| 구성원 목록 | `members[].roleId` | string | 현재 부여된 역할 식별 |
| 구성원 목록 | `members[].roleName` | string | 대표 관리자, 상품 담당자 등 역할명 표시 |
| 구성원 목록 | `members[].status` | enum | 초대 대기, 활성, 이용 중지 상태 표시 |
| 구성원 목록 | `members[].invitedAt` | datetime? | 초대 발송 시각 표시 |
| 구성원 목록 | `members[].lastAccessedAt` | datetime? | 마지막 접속 시각 표시 |
| 역할 목록 | `roles[].roleId` | string | 권한 표에서 역할 식별 |
| 역할 목록 | `roles[].name` | string | 역할명 표시 |
| 역할 목록 | `roles[].description` | string | 역할의 책임 범위 설명 |
| 권한 표 | `permissions[].permissionId` | string | 메뉴·작업 권한 항목 식별 |
| 권한 표 | `permissions[].menuKey` | enum | 드롭, 상품, 주문, 정산 등 대상 메뉴 구분 |
| 권한 표 | `permissions[].actionKey` | enum | 조회, 등록, 수정, 승인 요청, 다운로드 구분 |
| 권한 표 | `rolePermissions[].accessLevel` | enum | 역할별 전체 허용, 일부 허용, 접근 불가 상태 표시·입력 |
| 구성원 초대 | `invite.email` | email | 새 구성원의 로그인 계정 입력 |
| 구성원 초대 | `invite.roleId` | string | 초대할 구성원에게 부여할 역할 선택 |
| 구성원 초대 | `invite.expiresAt` | datetime? | 초대 링크 만료 시각 표시 |
| 변경 이력 | `auditLogs[].actorDisplayName` | string | 권한을 변경한 구성원명 표시 |
| 변경 이력 | `auditLogs[].targetDisplayName` | string | 권한이 변경된 구성원명 표시 |
| 변경 이력 | `auditLogs[].beforeRoleName` | string | 변경 전 역할명 표시 |
| 변경 이력 | `auditLogs[].afterRoleName` | string | 변경 후 역할명 표시 |
| 변경 이력 | `auditLogs[].occurredAt` | datetime | 권한 변경 시각 표시 |
| 작업 상태 | `actions.canInviteMember` | boolean | 구성원 초대 가능 여부 결정 |
| 작업 상태 | `actions.canChangeRole` | boolean | 역할·권한 변경 가능 여부 결정 |
| 작업 상태 | `actions.canDeactivateMember` | boolean | 운영자 계정 비활성화 가능 여부 결정 |

## 관련 문서

- [판매자 사이트맵](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- [판매자 요구사항](../../00-requirements/REQ_A_03_seller.md)
- [판매자 드롭 운영 유스케이스](../../30-uc/UC_A_02_seller_manage_drop.md)
