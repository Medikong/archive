---
id: SD.A.20010
title: Context 판매자 도메인 모델 인덱스
type: service-design-domain-model-index
status: draft
tags: [service-design, seller, domain-model]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
bounded_context: BC.A.200
---

# Context 판매자 도메인 모델 인덱스

이 README는 탐색용 인덱스다. 상세 불변조건과 상태 전이는 책임 문서에서 관리한다.

바운디드 컨텍스트의 논리 소유권과 MSA 배포 소유권을 구분한다. 현재 배치는 다음과 같으며 소유 미확정 Aggregate는 구현 대상으로 삼지 않는다.

| Aggregate·모델 | 목표 배포 단위 | 상태 |
| --- | --- | --- |
| `SellerAccount`, `StoreProfile`, `SellerTeam`, `SellerMembership`, `SellerRole` | 미확정 | Auth에 배정하지 않음 |
| `SellerProduct`, `DropProposal`, `DropReview` | `catalog-service` 후보 | seller 계약 미구현 |
| `OrderExport` | `order-service` 후보 | seller 주문·export 계약 미구현 |
| `SellerIssue` | 미확정 | 플랫폼 운영 case 계약 대기 |

| 문서 | Aggregate·모델 | Command·Read Model |
| --- | --- | --- |
| [판매자 관리](seller-management.md) | `SellerAccount`, `StoreProfile`, `SellerTeam`, `SellerMembership`, `SellerRole` | `CMD.A.200-01~05`, `RM.A.200-01~02` |
| [상품과 드롭 제안](seller-proposal.md) | `SellerProduct`, `DropProposal`, `DropReview` | `CMD.A.200-06~11`, `RM.A.200-03~04` |
| [주문 자료와 운영 이슈](seller-operations.md) | `OrderExport`, `SellerIssue` | `CMD.A.200-12~17`, `RM.A.200-07`, `RM.A.200-10` |
| [Read Model](read-models.md) | 판매자 전용 투영 모델 | `RM.A.200-01~10` |

공통 식별자는 opaque string을 사용하고 Aggregate는 0부터 증가하는 `version`을 가진다. 외부 원본은 `ExternalRef`·`SnapshotRef`로만 보존하며 상품·주문·결제·승인 원문을 복제하지 않는다. Aggregate 사이에 서비스 간 FK나 공유 트랜잭션을 만들지 않는다.
