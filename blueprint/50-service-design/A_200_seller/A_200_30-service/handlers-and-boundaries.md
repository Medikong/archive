---
id: SD.A.20030.HANDLERS
title: 판매자 Handler와 책임 경계
type: service-design-service
status: draft
tags: [service-design, seller, handler, transaction]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
service: SD.A.20030
---

# 판매자 Handler와 책임 경계

## MSA 배치

| Command·Read Model | 목표 배포 단위 | 상태 |
| --- | --- | --- |
| `CMD.A.200-01~05`, `RM.A.200-01~02` | 소유 서비스 미확정 | SellerAccount·StoreProfile·membership·role 원장을 Auth에 넣지 않음 |
| `CMD.A.200-06~11`, `RM.A.200-03~04`, `RM.A.200-06` | `catalog-service` 배정 후보 | seller 상품·제안·검수 계약 미구현 |
| `RM.A.200-05`, `RM.A.200-08~09` | 소유 서비스 미확정 | 교차 서비스 집계와 settlement 원천 계약 대기 |
| `CMD.A.200-12~14`, `RM.A.200-07` | `order-service` 배정 후보 | seller 주문·export 계약 미구현 |
| `CMD.A.200-15~17`, `RM.A.200-10` | 소유 서비스 미확정 | 플랫폼 운영 case 계약 대기 |

아래 Handler 이름은 도메인 책임을 추적하기 위한 논리 이름이다. 목표 배포 단위가 `소유 서비스 미확정`인 항목은 코드 생성이나 route 공개의 근거로 사용할 수 없다.

## Command Handler

| Command | Handler | Aggregate | 트랜잭션 결과 |
| --- | --- | --- | --- |
| `CMD.A.200-01` | `UpdateSellerAccountHandler` | `SellerAccount` | account·version·audit·`EVT.A.200-01` |
| `CMD.A.200-02` | `UpdateStoreProfileHandler` | `StoreProfile` | profile·version·audit·`EVT.A.200-02` |
| `CMD.A.200-03` | `InviteSellerMemberHandler` | `SellerTeam` | invitation·team version·audit·`EVT.A.200-03` |
| `CMD.A.200-04` | `ChangeSellerRoleHandler` | `SellerTeam` | membership·permission version·audit·`EVT.A.200-04` |
| `CMD.A.200-05` | `DeactivateSellerMemberHandler` | `SellerTeam` | inactive 상태·permission version·audit·`EVT.A.200-05` |
| `CMD.A.200-06` | `SaveSellerProductHandler` | `SellerProduct` | product version·audit·`EVT.A.200-06` |
| `CMD.A.200-07` | `SaveDropProposalHandler` | `DropProposal` | immutable proposal version·`EVT.A.200-07` |
| `CMD.A.200-08` | `SubmitDropReviewHandler` | `DropProposal` | submit version 잠금·`EVT.A.200-08` |
| `CMD.A.200-09` | `ApplyDropReviewResultHandler` | `DropProposal` | decision ref·상태·`EVT.A.200-09` |
| `CMD.A.200-10` | `RequestApprovedProposalChangeHandler` | `DropProposal` | change request·`EVT.A.200-10` |
| `CMD.A.200-11` | `HandoffApprovedProposalHandler` | `DropProposal` | handoff ref·`EVT.A.200-11` |
| `CMD.A.200-12` | `RequestOrderExportHandler` | `OrderExport` | 요청 snapshot·audit·`EVT.A.200-12` |
| `CMD.A.200-13` | `CompleteOrderExportHandler` | `OrderExport` | object ref·checksum·`EVT.A.200-13` |
| `CMD.A.200-14` | `ExpireOrderExportHandler` | `OrderExport` | 만료·audit·`EVT.A.200-14` |
| `CMD.A.200-15` | `CreateSellerIssueHandler` | `SellerIssue` | issue·audit·`EVT.A.200-15` |
| `CMD.A.200-16` | `AddSellerIssueInformationHandler` | `SellerIssue` | timeline entry·`EVT.A.200-16` |
| `CMD.A.200-17` | `ApplySellerIssueResultHandler` | `SellerIssue` | 외부 result ref·`EVT.A.200-17` |

## Query Handler

| Read Model | Handler | 정책 |
| --- | --- | --- |
| `RM.A.200-01` | `GetSellerSettingsHandler` | SellerAccount·StoreProfile 원장과 현재 membership 재검증 |
| `RM.A.200-02` | `GetSellerAccessContextHandler`, `ListSellerMembersHandler` | SellerTeam 원장과 permission version 재검증 |
| `RM.A.200-03` | `ListSellerProductsHandler` | SellerProduct 원장과 seller ownership 재검증 |
| `RM.A.200-04` | `GetDropProposalHandler` | DropProposal 원장과 검수 projection freshness 보존 |
| `RM.A.200-05` | `GetSellerDashboardHandler` | 투영된 section snapshot만 조합하고 section freshness 보존 |
| `RM.A.200-06` | `ListSellerDropsHandler` | proposal·공개 Drop 투영 상태와 원천 watermark 보존 |
| `RM.A.200-07` | `ListSellerOrdersHandler` | 마스킹된 Order projection과 export 상태만 조회 |
| `RM.A.200-08` | `GetSellerAnalyticsHandler` | metric bucket의 추정·확정·watermark 보존 |
| `RM.A.200-09` | `GetSellerSettlementsHandler` | 외부 Settlement snapshot과 unavailable 상태 보존 |
| `RM.A.200-10` | `ListSellerIssuesHandler` | SellerIssue 원장과 외부 case 결과 timeline 조합 |

## Port

| Port | 책임 | 실패 원칙 |
| --- | --- | --- |
| `AuthContextPort` | Auth `API.A.300-16`에서 검증된 `user_id`·Session ref 획득 | 익명/장애를 seller membership으로 추측하지 않음 |
| `ReauthenticationProofPort` | seller purpose proof의 audience·purpose·session binding 검증 | proof 원문 저장 금지, 불일치는 403 |
| `ModerationDecisionPort` | 검수 decision ref·version 검증 | gap이 닫히기 전 성공 callback 계약 배포 금지 |
| `DropHandoffPort` | 승인 proposal snapshot 전달 | 공개 Drop 생성 성공을 로컬에서 추측하지 않음 |
| `OrderExportSourcePort` | `order-service`가 소유한 `RM.A.200-07` snapshot에서 허용 field 읽기 | 다른 서비스가 buyer API나 Order DB를 직접 조회하지 않음 |
| `ObjectStoragePort` | private export object 업로드·삭제·signed URL | object 성공만으로 `READY` 확정 금지 |
| `OperationCasePort` | SellerIssue 외부 case 전달·결과 ref 검증 | penalty·정산 상태를 Seller가 만들지 않음 |
| `CouponSellerPort` | Coupon seller API와 scope 계약 | seller별 목록·성과 계약 미구현은 typed unavailable |
