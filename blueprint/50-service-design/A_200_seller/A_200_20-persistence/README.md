---
id: SD.A.20020
title: Context 판매자 영속성 설계 인덱스
type: service-design-persistence-index
status: draft
tags: [service-design, seller, persistence, postgres, msa]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
---

# Context 판매자 영속성 설계 인덱스

| 문서 | 내용 |
| --- | --- |
| [쓰기 원장](write-models.md) | 논리 table, unique·FK·version 제약, 목표 서비스별 Repository 경계 |
| [Read Model과 투영](read-models-and-projections.md) | `RM.A.200-01~10`, 목표 소유 서비스, inbox, watermark, 재구성 |
| [신뢰성·감사·파일](reliability-audit-and-files.md) | 서비스별 idempotency·outbox·감사, Order export object 수명주기 |

## 물리 저장소 결론

- 공유 `seller` 데이터베이스를 새로 만들지 않는다.
- 각 Aggregate는 실제 소유 서비스의 PostgreSQL에 저장한다. `catalog-service` 후보 Aggregate는 Catalog DB, `OrderExport` 후보 Aggregate는 Order DB에 속한다.
- `SellerAccount`, `StoreProfile`, `SellerMembership`, `SellerRole`, 통합 조회 모델과 `SellerIssue`는 소유 서비스가 미확정이므로 물리 table과 migration을 만들 수 없다.
- 아래 table 이름은 schema 검토용 논리 후보다. 소유 서비스가 정해지기 전에는 공통 DB나 별도 Seller DB를 뜻하지 않는다.
- Redis, broker와 object storage 결과만으로 Command 성공을 확정하지 않는다. 실제 소유 서비스의 PostgreSQL 트랜잭션이 version과 멱등 결과의 최종 판단 근거다.

## Command·Read Model 저장 책임

| 식별자 | 목표 물리 소유자 | 상태 |
| --- | --- | --- |
| `CMD.A.200-01~05`, `RM.A.200-01~02` | 미확정 서비스의 PostgreSQL | 구현 대기 |
| `CMD.A.200-06~11`, `RM.A.200-03~04`, `RM.A.200-06` | `catalog-service` PostgreSQL 후보 | seller schema·migration 미구현 |
| `CMD.A.200-12~14`, `RM.A.200-07` | `order-service` PostgreSQL과 private object storage 후보 | seller export schema·migration 미구현 |
| `RM.A.200-05`, `RM.A.200-08~09` | 미확정 조회 모델 소유 서비스 | 교차 서비스 projection 구현 금지 |
| `CMD.A.200-15~17`, `RM.A.200-10` | 미확정 서비스의 PostgreSQL | 플랫폼 운영 case 계약 대기 |

서비스 간 FK와 공유 트랜잭션은 만들지 않는다. 다른 서비스의 식별자는 opaque reference와 원천 version으로 저장하고, 후속 작업은 outbox Event 또는 명시적 API로 연결한다.
