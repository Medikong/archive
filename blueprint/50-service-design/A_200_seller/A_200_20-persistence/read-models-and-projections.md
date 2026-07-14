---
id: SD.A.20020.READ
title: 판매자 Read Model과 Event 투영
type: service-design-persistence
status: draft
tags: [service-design, seller, read-model, projection, inbox]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
persistence: SD.A.20020
---

# 판매자 Read Model과 Event 투영

## 물리 소유 상태

| Read Model | 목표 소유 서비스 | 상태 |
| --- | --- | --- |
| `RM.A.200-01~02` | 미확정 | table 생성 금지 |
| `RM.A.200-03~04`, `RM.A.200-06` | `catalog-service` 후보 | seller 계약 미구현 |
| `RM.A.200-07` | `order-service` 후보 | seller 주문·마스킹 계약 미구현 |
| `RM.A.200-05`, `RM.A.200-08~10` | 미확정 | 교차 서비스 projection 또는 외부 원천 소유자 필요 |

아래 table은 논리 후보이며 한 데이터베이스에 함께 생성하지 않는다.

## 저장 구조

| table 후보 | Read Model | index 기준 |
| --- | --- | --- |
| `seller_settings_view` | `RM.A.200-01` | `seller_id` |
| `seller_team_view`, `seller_permission_matrix_view` | `RM.A.200-02` | `(seller_id, status)`, `(seller_id, user_id)` |
| `seller_product_workspace_view` | `RM.A.200-03` | `(seller_id, updated_at desc)`, option search key |
| `seller_proposal_review_view` | `RM.A.200-04` | `(seller_id, state, updated_at desc)` |
| `seller_dashboard_sections` | `RM.A.200-05` | `(seller_id, section_key)` |
| `seller_drop_status_view` | `RM.A.200-06` | `(seller_id, drop_state, opens_at)` |
| `seller_order_fulfillment_view` | `RM.A.200-07` | `(seller_id, order_state, occurred_at desc)`, `(seller_id, drop_id)` |
| `seller_sales_metric_buckets` | `RM.A.200-08` | `(seller_id, bucket_start, dimension_type, dimension_key)` |
| `seller_settlement_view` | `RM.A.200-09` | `(seller_id, settlement_period, state)` |
| `seller_issue_view` | `RM.A.200-10` | `(seller_id, state, updated_at desc)` |

## 투영 규칙

- 소비자는 `(consumer_name, event_id)` inbox unique를 먼저 확보하고 `(source_context, aggregate_id, aggregate_version)` 순서를 검사한다.
- 같은 Event 중복은 성공적으로 무시하되, 같은 ID의 다른 payload hash는 손상된 계약으로 격리한다.
- version gap은 해당 aggregate partition을 stale로 표시하고 원천 replay/snapshot 복구 전까지 후속 version을 확정 지표에 반영하지 않는다.
- 지표 bucket은 원천 event ID와 contribution ledger를 보존해 같은 Event 집합으로 재구성할 수 있어야 한다.
- Query는 여러 원천을 요청 시점에 fan-out하지 않는다. 실제 소유 서비스가 정해진 뒤 그 서비스가 Event로 미리 만든 section snapshot만 조합한다.
- projection 지연이나 원천 장애는 `stale`, `partial`, `unavailableSections`로 공개하며 0·빈 목록으로 정상화하지 않는다.

## 재구성

새 schema version은 shadow table에 replay하고 event count·watermark·checksum을 대조한 뒤 read alias를 전환한다. 전체 재구성 중 기존 view는 마지막 정상 `asOf`와 stale 표시로 계속 제공할 수 있지만, 개인정보 삭제·접근 철회처럼 즉시 차단이 필요한 항목은 query gate에서 현재 membership·policy를 다시 검사한다.
