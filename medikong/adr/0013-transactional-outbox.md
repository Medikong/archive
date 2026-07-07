---
id: ADR-0013
title: "주문과 결제 이벤트에 transactional outbox를 사용한다"
status: proposed
date: 2026-07-02
areas:
  - events
  - reliability
  - data
repos:
  - workspaces
  - services
decision_drivers:
  - dual write failure 회피
  - 재시도 가능한 발행
  - 멱등 consumer
related:
  - ../06-event-contracts.md
  - ../04-data-design.md
links: []
supersedes: []
superseded_by: null
---

# ADR 0013: 주문과 결제 이벤트에 transactional outbox를 사용한다

## 상태

Proposed

## 배경

주문 또는 결제 DB 변경 후 Kafka publish가 실패하면 서비스 간 상태가 어긋난다. 반대로 Kafka publish 후 DB commit이 실패해도 잘못된 이벤트가 나갈 수 있다.

## 결정

`order-service`와 `payment-service`는 DB 변경과 outbox row 생성을 같은 transaction에 넣는다. Kafka publish는 outbox relay가 재시도 가능하게 수행한다.

## 대안

| 대안 | 장점 | 단점 | 판단 |
| --- | --- | --- | --- |
| API handler에서 DB commit 후 Kafka publish | 구현이 단순 | dual write failure에 취약 | 기각 |
| distributed transaction | 이론상 강한 일관성 | 운영 복잡도 과다 | 기각 |
| transactional outbox | 재시도 가능, 구현 가능 | relay와 outbox lag 운영 필요 | 채택 |

## 결과

좋아지는 점:

- 이벤트 발행 실패를 재시도할 수 있다.
- consumer idempotency와 DLQ 운영 기준이 명확해진다.
- payment/order saga가 관측 가능해진다.

비용:

- outbox table, relay worker, metrics가 필요하다.
- event schema versioning을 관리해야 한다.

## 후속 작업

| 상태 | 작업 | 담당 | 연결 문서 |
| --- | --- | --- | --- |
| 계획됨 | outbox schema 추가 | 미지정 | ../04-data-design.md |
| 계획됨 | event envelope 구현 | 미지정 | ../06-event-contracts.md |
| 계획됨 | outbox lag dashboard 추가 | 미지정 | ../09-observability-slo.md |
