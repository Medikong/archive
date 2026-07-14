---
id: SD.A.20020.WRITE
title: Context 판매자 PostgreSQL 쓰기 원장
type: service-design-persistence
status: draft
tags: [service-design, seller, postgres, write-model]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
persistence: SD.A.20020
---

# Context 판매자 PostgreSQL 쓰기 원장

이 문서의 table은 하나의 Seller DB schema가 아니라 논리 후보 목록이다. `seller_products`부터 `proposal_change_requests`까지는 `catalog-service` DB 배정 후보이고, `order_exports`는 `order-service` DB 배정 후보다. 계정·스토어·membership·role과 issue table은 소유 서비스가 정해질 때까지 migration 대상으로 삼지 않는다.

## table 소유권

| table 후보 | PK·핵심 unique | 역할 |
| --- | --- | --- |
| `seller_accounts` | `seller_id`, business ref 조건부 unique | SellerAccount 원장, seller type·verification snapshot·status·version |
| `store_profiles` | `seller_id`, 공개 slug 조건부 unique | StoreProfile 원장·version |
| `seller_roles` | `(seller_id, role_id)`, `(seller_id, normalized_name)` | role과 permission set·version |
| `seller_memberships` | `membership_id`, `(seller_id, user_id)` active 조건부 unique | membership·role·status·permission_version |
| `seller_member_invitations` | `invitation_id`, `(seller_id, user_id)` pending 조건부 unique | 초대 제안 상태. 이메일 미저장 |
| `seller_products` | `product_id`, `(seller_id, external_ref)` 조건부 unique | 상품 원본·version |
| `seller_product_options` | `(product_id, option_id)`, seller FK | option·공급 선언 |
| `drop_proposals` | `proposal_id`, `(seller_id, client_ref)` | draft/submission/review/handoff 상태·version |
| `drop_proposal_versions` | `(proposal_id, version)` | canonical immutable snapshot과 hash |
| `drop_reviews` | `review_id`, `external_decision_id` unique | 제출 version과 decision ref |
| `proposal_change_requests` | `change_request_id`, `(proposal_id, requested_version)` | 승인 후 변경 diff·상태 |
| `order_exports` | `export_id`, `(seller_id, idempotency_business_ref)` | 요청 filter·watermark·object ref·만료·version |
| `seller_issues` | `issue_id`, `(seller_id, client_ref)` | 이슈 원장·external case ref·version |
| `seller_issue_entries` | `entry_id`, `(issue_id, sequence)` | 추가 정보·외부 결과 timeline |

## 공통 column과 제약

- 변경 가능한 Aggregate는 `version bigint not null`, `created_at`, `updated_at`을 가진다. update는 `WHERE id=? AND seller_id=? AND version=?` 후 version을 1 증가시킨다.
- JSONB는 외부 schema를 숨기거나 무제한 payload를 저장하는 용도로 쓰지 않는다. permission set·canonical diff처럼 versioned schema가 있는 값에만 사용한다.
- `seller_id`는 자식 table의 FK·index 선두에 둔다. 애플리케이션 필터만으로 tenant 격리를 가정하지 않는다.
- 상태·reason code는 CHECK 또는 참조 정책 version으로 제한한다. 자유 문자열을 업무 상태로 사용하지 않는다.
- 민감 원문, signed URL, Auth credential, 쿠폰 코드, 주문 전체 payload는 저장하지 않는다.

## Repository와 트랜잭션

- Aggregate Repository는 다른 Aggregate를 같은 save 호출로 갱신하지 않는다.
- Aggregate, state transition ledger, audit entry, idempotency result, outbox는 해당 Aggregate를 소유한 서비스의 PostgreSQL 트랜잭션 하나에 저장한다.
- 외부 API 호출 중 DB 트랜잭션을 열어 두지 않는다. 외부 snapshot을 먼저 얻고 저장 직전에 version/hash를 다시 검증한다.
- `CMD.A.200-13`, `CMD.A.200-17` 같은 worker callback은 external result ID unique와 expected state/version을 함께 검사한다.
