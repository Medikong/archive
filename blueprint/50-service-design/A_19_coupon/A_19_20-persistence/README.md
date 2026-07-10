---
id: SD.A.1920
title: Context 쿠폰 영속성 설계 인덱스
type: service-design-persistence-index
status: draft
tags: [service-design, coupon, persistence, index]
source: local
created: 2026-07-09
updated: 2026-07-10
service_design: SD.A.19
bounded_context: BC.A.19
domain_model: SD.A.1910
---

# Context 쿠폰 영속성 설계 인덱스

## 역할

Context 쿠폰의 Postgres 쓰기 모델, 원장·멱등·outbox/inbox, 조회 모델과 인덱스를 책임별 문서로 안내한다. 스키마와 제약의 상세 정의는 하위 문서에서만 관리한다.

## 원천

- [BC.A.19 Context 쿠폰](../../../40-event-storming-bounded-context/BC_A_19_coupon.md)
- [REQ.A.02 쿠폰 및 혜택](../../../00-requirements/REQ_A_02_coupon_benefit.md)
- [SD.A.1910 도메인 모델](../A_19_10-domain-model/README.md)
- 보조 근거: [기존 엔티티 설계](../../../../../workspaces/docs/architecture/coupon-service/02-entity-design.md)

## 하위 문서

| 문서 | 책임 | 상태 |
| --- | --- | --- |
| [쓰기 모델](write-models.md) | Campaign·Code·IssueRequest·UserCoupon·Redemption 저장 모델과 제약 | draft |
| [원장과 신뢰성](ledgers-and-reliability.md) | 수량 예약, 멱등성, append-only 원장, outbox/inbox, 복구와 트랜잭션 | draft |
| [조회 모델과 인덱스](read-models-and-indexes.md) | 조회 투영, partial unique/index, Redis/MQ 정합성, 보존·마이그레이션 | draft |

## 결정 경계

- Postgres 원장과 제약이 최종 업무 상태를 보장한다. outbox는 원장 변경과 같은 트랜잭션에서 기록한다.
- Redis는 선행 차단과 캐시, MQ는 전달과 부하 분산만 담당한다.
- Aggregate별 저장은 별도 트랜잭션이며 후속 저장은 Event와 Policy가 새 Command로 수행한다.
- 외부 Context 원본은 저장하지 않고 식별자, 기준 시각, 버전, 해시가 있는 스냅샷만 보존한다.
- 실제 DDL과 마이그레이션 파일은 이 설계의 범위가 아니다.
