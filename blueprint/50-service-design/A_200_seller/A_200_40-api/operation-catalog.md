---
id: API.A.200.CATALOG
title: Context 판매자 API operation catalog
type: service-design-api-catalog
status: draft
tags: [service-design, seller, api, operation-catalog]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
api_design: SD.A.20040
---

# Context 판매자 API operation catalog

## 사용 조건

이 catalog의 API ID와 method/path는 도메인 추적성과 wire 검토를 위한 논리 초안이다. 하나의 Seller API 서버에 배포하지 않는다. 목표 배치는 다음과 같으며, 실제 path와 인증 방식은 해당 서비스 OpenAPI와 Ingress route에서 다시 확정한다.

| API | 목표 소유 서비스 | 상태 |
| --- | --- | --- |
| `API.A.200-01~08` | 미확정 | 보호 route 공개 불가 |
| `API.A.200-09~16`, `API.A.200-18` | `catalog-service` 후보 | seller 계약 미구현 |
| `API.A.200-19~22` | `order-service` 후보 | seller 계약 미구현 |
| `API.A.200-17`, `API.A.200-23~28` | 미확정 | 통합 조회·정산·운영 case 소유자 필요 |

따라서 아래 `/api/v1/internal` path는 canonical 배포 path가 아니다. 현재 OpenAPI 초안과 항목을 대조하기 위해 유지한다.

## Access·Seller Management

| API ID | Method / Path | Command·RM | 권한 | request 핵심 | response 핵심 | 동시성 |
| --- | --- | --- | --- | --- | --- | --- |
| `API.A.200-01` | `GET /api/v1/internal/sellers/{sellerId}/access-context` | `RM.A.200-02` access slice | workload + Auth `user_id`, target seller membership | Auth subject·Session ref는 credential에서, path `sellerId` | membership ID/status/version, permission version·set, seller summary. 개인정보 없음 | `ETag` permission version |
| `API.A.200-02` | `GET /api/v1/internal/sellers/{sellerId}/settings` | `RM.A.200-01` | `seller.account.read`, `seller.store.read` | include preview 선택 | SellerAccount·StoreProfile·verification snapshot ref | `ETag` composite version |
| `API.A.200-03` | `PUT /api/v1/internal/sellers/{sellerId}/account` | `CMD.A.200-01` | `seller.account.write`, 민감 변경은 `seller_account_update` proof | seller type, 책임 고지, 고객 안내, verification ref | account ID·version·changed fields | 멱등키 + `If-Match` |
| `API.A.200-04` | `PUT /api/v1/internal/sellers/{sellerId}/store-profile` | `CMD.A.200-02` | `seller.store.write` | name, asset refs, introduction, buyer notice | profile ID·version·preview ref | 멱등키 + `If-Match` |
| `API.A.200-05` | `GET /api/v1/internal/sellers/{sellerId}/members` | `RM.A.200-02` | `seller.member.read` | status·cursor | membership, role, permission set, audit summary | `ETag` team version |
| `API.A.200-06` | `POST /api/v1/internal/sellers/{sellerId}/member-invitations` | `CMD.A.200-03` | owner + `seller.member.write`, `seller_member_manage` proof | `userId`, role ID, reason ref | invitation ID·status·team version | 멱등키 + team `If-Match` |
| `API.A.200-07` | `PUT /api/v1/internal/sellers/{sellerId}/members/{membershipId}/role` | `CMD.A.200-04` | owner + `seller.role.permission.write`, `seller_member_manage` proof | role ID, reason ref | membership·permission version | 멱등키 + membership `If-Match` |
| `API.A.200-08` | `POST /api/v1/internal/sellers/{sellerId}/members/{membershipId}/deactivation` | `CMD.A.200-05` | owner + `seller.member.write`, `seller_member_manage` proof | reason ref | membership status/version | 멱등키 + membership `If-Match` |

## Seller Proposal

| API ID | Method / Path | Command·RM | 권한 | request 핵심 | response 핵심 | 동시성 |
| --- | --- | --- | --- | --- | --- | --- |
| `API.A.200-09` | `GET /api/v1/internal/sellers/{sellerId}/products` | `RM.A.200-03` | `seller.product.read` | status·search·cursor | seller 상품·option·공급 선언 요약 | `ETag`, cursor snapshot |
| `API.A.200-10` | `PUT /api/v1/internal/sellers/{sellerId}/products/{productId}` | `CMD.A.200-06` | `seller.product.write` | media refs, description, price proposal, options, supply | product ID·version·validation summary | 멱등키 + `If-Match`; create는 `If-None-Match: *` |
| `API.A.200-11` | `GET /api/v1/internal/sellers/{sellerId}/drop-proposals/{proposalId}` | `RM.A.200-04` | `seller.drop.read` | include review/history | draft·submitted version, decision refs, before/after | `ETag` proposal version |
| `API.A.200-12` | `PUT /api/v1/internal/sellers/{sellerId}/drop-proposals/{proposalId}` | `CMD.A.200-07` | `seller.drop.write` | product ref, schedule, quantity, limit, shipping, return | proposal ID·draft version·completeness | 멱등키 + `If-Match`; create는 `If-None-Match: *` |
| `API.A.200-13` | `POST /api/v1/internal/sellers/{sellerId}/drop-proposals/{proposalId}/review-submissions` | `CMD.A.200-08` | `seller.drop.review.submit` | submit version, evidence refs | review request ID·`SUBMITTED`·version | 멱등키 + proposal `If-Match` |
| `API.A.200-14` | `POST /api/v1/internal/sellers/{sellerId}/drop-proposals/{proposalId}/review-results` | `CMD.A.200-09` | moderation workload | decision ID, submitted version, decision, reason ref, decidedAt | review ID·proposal state/version | 멱등키 + external decision unique + `If-Match` |
| `API.A.200-15` | `POST /api/v1/internal/sellers/{sellerId}/drop-proposals/{proposalId}/change-requests` | `CMD.A.200-10` | `seller.drop.write`, 위험 변경은 approval ref | approved version, canonical diff, reason ref | change request ID·state·proposal version | 멱등키 + proposal `If-Match` |
| `API.A.200-16` | `POST /api/v1/internal/sellers/{sellerId}/drop-proposals/{proposalId}/handoffs` | `CMD.A.200-11` | drop handoff worker | approved version, decision ref | handoff ID·state·external ref | 멱등키 + approved version unique |

## Operations Query·Order Export·Issue

| API ID | Method / Path | Command·RM | 권한 | request 핵심 | response 핵심 | 동시성 |
| --- | --- | --- | --- | --- | --- | --- |
| `API.A.200-17` | `GET /api/v1/internal/sellers/{sellerId}/dashboard` | `RM.A.200-05` | `seller.dashboard.read` | 기간·section 선택 | 우선 작업, KPI, 진행 drop, 최근 주문, 일정 + section freshness | `ETag` projection snapshot |
| `API.A.200-18` | `GET /api/v1/internal/sellers/{sellerId}/drops` | `RM.A.200-06` | `seller.drop.read` | proposal/public state·기간·cursor | proposal와 공개 drop 상태 대조, unavailable section | `ETag`, cursor snapshot |
| `API.A.200-19` | `GET /api/v1/internal/sellers/{sellerId}/orders` | `RM.A.200-07` | `seller.order.read` | 상태·drop·기간·cursor | 마스킹 주문·출고 정보와 export 상태 | `ETag`, cursor snapshot |
| `API.A.200-20` | `POST /api/v1/internal/sellers/{sellerId}/order-exports` | `CMD.A.200-12` | `seller.order.export`, `seller_order_export` proof | purpose, filter, format, policy ref | `202`, export ID·`REQUESTED`, status path | 멱등키 + source watermark |
| `API.A.200-21` | `POST /api/v1/internal/sellers/{sellerId}/order-exports/{exportId}/completion` | `CMD.A.200-13` | export worker | object ref, checksum, row count, watermark, expiresAt | export ID·`READY`·version | 멱등키 + export `If-Match` |
| `API.A.200-22` | `POST /api/v1/internal/sellers/{sellerId}/order-exports/{exportId}/expiration` | `CMD.A.200-14` | expiry worker | expiredAt, reason ref | export ID·`EXPIRED`·version | 멱등키 + export `If-Match` |
| `API.A.200-23` | `GET /api/v1/internal/sellers/{sellerId}/analytics` | `RM.A.200-08` | `seller.analytics.read` | 기간, product·option·drop·source·coupon dimension | metric buckets, estimate/final, `asOf`, watermark | `ETag` metric snapshot |
| `API.A.200-24` | `GET /api/v1/internal/sellers/{sellerId}/settlements` | `RM.A.200-09` | `seller.settlement.read` | 기간·상태·cursor | 외부 settlement snapshot과 Coupon cost refs | `ETag`; gap이면 partial/unavailable |
| `API.A.200-25` | `GET /api/v1/internal/sellers/{sellerId}/issues` | `RM.A.200-10` | `seller.issue.read` | 상태·category·cursor | issue·timeline·external case summary | `ETag`, cursor snapshot |
| `API.A.200-26` | `POST /api/v1/internal/sellers/{sellerId}/issues` | `CMD.A.200-15` | `seller.issue.write` | category, reason code, subject refs, description, evidence refs | issue ID·`OPEN`·version | 멱등키 + client ref unique |
| `API.A.200-27` | `POST /api/v1/internal/sellers/{sellerId}/issues/{issueId}/information` | `CMD.A.200-16` | `seller.issue.write` | reason ref, description, evidence refs | entry ID·issue state/version | 멱등키 + issue `If-Match` |
| `API.A.200-28` | `POST /api/v1/internal/sellers/{sellerId}/issues/{issueId}/results` | `CMD.A.200-17` | platform operations workload | external result ID, case ref, public status/summary, occurredAt | issue state/version·result ref | 멱등키 + result unique + `If-Match` |

## seller scope와 재검증

- 목표 구조에서 브라우저는 Ingress를 통해 실제 소유 서비스의 seller API를 호출한다. BFF가 seller scope를 만들어 전달하지 않는다.
- Ingress가 전달한 인증 context는 필요조건일 뿐 권한 원장이 아니다.
- 모든 operation은 path `sellerId`, 현재 membership seller ID와 리소스 소유 seller ID가 같은지 확인한다.
- Query도 current membership status와 permission version을 확인한다. membership 저장소 장애 시 stale 권한으로 허용하지 않는다.
- `API.A.200-14/16/21/22/28`은 seller user scope 대신 제한된 workload scope를 사용하지만 resource seller ID는 동일하게 확인한다.

## 공통 response와 오류

Query는 `data`와 `meta.requestId`, `meta.asOf`, `meta.stale`, `meta.partial`, `meta.unavailableSections`, `meta.sourceWatermarks`를 분리한다. Command는 resource ID, `status`, `version`, `operationId`를 반환한다.

모든 operation은 입력 오류 `400`, 인증 `401`, 권한 `403`, 동일 존재 숨김 `404`, 멱등·version·상태 `409`, 업무 규칙 `422`, 필수 dependency·projection `503`을 사용한다. `API.A.200-20`은 접수 뒤 완료를 보장하지 않으며 상태 조회는 `RM.A.200-07`/`API.A.200-19`에서 확인한다.
