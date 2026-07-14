---
id: API.A.200.EVENTS
title: Context 판매자 Event 계약
type: service-design-event-contract
status: draft
tags: [service-design, seller, event, outbox, inbox]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
api_design: SD.A.20040
bounded_context: BC.A.200
---

# Context 판매자 Event 계약

## 발행 서비스 배치

| Event | 목표 발행 서비스 | 상태 |
| --- | --- | --- |
| `EVT.A.200-01~05` | 미확정 | Seller Management 소유 서비스 결정 전 발행 불가 |
| `EVT.A.200-06~11` | `catalog-service` 후보 | seller Aggregate·outbox 미구현 |
| `EVT.A.200-12~14` | `order-service` 후보 | OrderExport·outbox 미구현 |
| `EVT.A.200-15~17` | 미확정 | Seller Issue·플랫폼 운영 case 소유자 필요 |

Event 번호가 하나의 topic, broker consumer 또는 Seller 서비스 배포를 뜻하지 않는다. 실제 발행 서비스가 확정되면 해당 서비스 계약에 event type과 schema를 편입한다.

## 공통 envelope

| 필드 | 규칙 |
| --- | --- |
| `event_id` | 전역 고유 ID. 재발행해도 바꾸지 않음 |
| `event_type` | `EVT.A.200-01~17`에 대응하는 안정적 유형 |
| `aggregate_type`, `aggregate_id`, `seller_id` | 원천 Aggregate와 tenant 범위 |
| `aggregate_version` | 상태 전이가 확정된 version |
| `occurred_at` | UTC 발생 시각 |
| `correlation_id`, `causation_id` | 최초 요청과 직접 원인 |
| `payload_schema_version` | payload 호환성 version |
| `data` | 결과 ID·변경 field·외부 ref만 포함. 원본 payload 복제 금지 |

## 발행 Event

| Event | Aggregate | 최소 payload | 주 소비자 |
| --- | --- | --- | --- |
| `EVT.A.200-01` | SellerAccount | seller ID, version, changed fields, verification snapshot ref | 설정 투영, 감사 |
| `EVT.A.200-02` | StoreProfile | seller ID, profile version, public asset refs | 설정·공개 store 투영 |
| `EVT.A.200-03` | SellerMembership | invitation ID, user ID, role ID, membership version | 팀 투영, 알림 gap |
| `EVT.A.200-04` | SellerMembership | membership ID, role ID, permission version | scope 무효화, 팀 투영 |
| `EVT.A.200-05` | SellerMembership | membership ID, inactiveAt, permission version | scope 무효화, 팀 투영 |
| `EVT.A.200-06` | SellerProduct | product ID, version, changed fields | 상품 workspace 투영 |
| `EVT.A.200-07` | DropProposal | proposal ID, draft version, completeness result | review workspace |
| `EVT.A.200-08` | DropProposal | review request ID, submitted version, snapshot ref | 플랫폼 검수 adapter |
| `EVT.A.200-09` | DropProposal | decision ID, submitted version, decision, reason ref | review 투영, handoff policy |
| `EVT.A.200-10` | DropProposal | change request ID, approved version, diff hash, reason ref | 플랫폼 검수 adapter |
| `EVT.A.200-11` | DropProposal | handoff ID, approved version, snapshot ref | Drop/Catalog adapter |
| `EVT.A.200-12` | OrderExport | export ID, purpose, filter hash, watermark | export worker, 감사 |
| `EVT.A.200-13` | OrderExport | export ID, object ref, checksum, row count, expiresAt | 주문 workspace 투영 |
| `EVT.A.200-14` | OrderExport | export ID, expiredAt, delete task ref | 다운로드 gate, 감사 |
| `EVT.A.200-15` | SellerIssue | issue ID, category, subject refs, evidence refs | 플랫폼 운영 adapter |
| `EVT.A.200-16` | SellerIssue | issue ID, entry ID, evidence refs | 플랫폼 운영 adapter |
| `EVT.A.200-17` | SellerIssue | issue ID, external result ID, case ref, public status | issue 투영 |

Event에는 이메일·휴대폰·role label, proof·token, 주문 개인정보, export signed URL, evidence 원문, 검수·운영 결과 원문을 넣지 않는다.

## 전달 보장

- Aggregate·감사·outbox는 해당 Aggregate 소유 서비스의 PostgreSQL 트랜잭션 하나에 저장한다.
- 발행은 최소 한 번이며 소비자는 `(consumer_name, event_id)` inbox와 업무 unique key를 모두 사용한다.
- 같은 ID의 다른 payload hash, 지원하지 않는 schema, aggregate version gap은 실패 원장에 남긴다.
- broker 종류·topic·partition·보존 기간은 이 설계에서 확정하지 않는다.

## 외부 수신 계약

| 소유 Context | 필요한 Event 사실 | 처리 대상 | 계약 상태 |
| --- | --- | --- | --- |
| 플랫폼 검수 | decision ID, review request ID, submitted version, decision·reason ref·decidedAt | `CMD.A.200-09` | 외부 계약 대기 |
| Drop/Catalog | handoff 수신 결과, published/opened/closed/blocked versioned 상태 | `RM.A.200-05~06`, `RM.A.200-08` | 외부 계약 대기 |
| Order | seller 귀속 item, order 상태, 최소 출고 snapshot, occurredAt/version | `RM.A.200-05`, `RM.A.200-07~08` | 외부 계약 대기 |
| Payment | order ref, seller 귀속 금액, approved/failed/canceled, version | `RM.A.200-05`, `RM.A.200-08` | 외부 계약 대기 |
| Coupon | seller campaign·issuance·redemption·revoke·cost attribution | `RM.A.200-05`, `RM.A.200-08~09` | 설계 보강·구현 대기 |
| Settlement | period, state, gross/net/hold/deduction refs, version | `RM.A.200-09` | 외부 계약 대기 |
| 플랫폼 운영 | case result ID, issue ID, public status/summary, penalty·settlement refs | `CMD.A.200-17`, `RM.A.200-10` | 외부 계약 대기 |

구체 schema가 없는 외부 Event는 AsyncAPI나 consumer 배포 계약으로 만들지 않는다. 현재 서비스 코드의 Event 이름은 조사 근거일 뿐 canonical seller projection 계약이 아니다.
