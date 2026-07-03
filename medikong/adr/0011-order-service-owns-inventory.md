---
id: ADR-0011
title: "order-service가 재고 예약을 소유한다"
status: proposed
date: 2026-07-02
areas:
  - data
  - order
  - reliability
repos:
  - workspaces
  - services
decision_drivers:
  - oversell 0
  - transaction 경계
  - 재시도 안전성
related:
  - ../04-data-design.md
  - ../07-critical-flows.md
links: []
supersedes: []
superseded_by: null
---

# ADR 0011: order-service가 재고 예약을 소유한다

## 상태

Proposed

## 배경

DropMong의 가장 중요한 불변 조건은 `oversell_count = 0`이다. 주문 생성과 재고 예약이 분리된 서비스에 있으면 동시성 제어, 보상 처리, retry 처리, 장애 복구가 모두 복잡해진다.

## 결정

`order-service`가 inventory bucket, reservation, order state를 함께 소유한다.

`catalog-service`는 상품과 드롭 공개 정보를 소유하지만, authoritative stock 판단은 하지 않는다.

## 대안

| 대안 | 장점 | 단점 | 판단 |
| --- | --- | --- | --- |
| catalog-service가 stock 포함 | 조회 모델이 단순하다 | cache와 진실 데이터가 섞인다 | 기각 |
| 별도 inventory-service | 책임 이름이 명확하다 | 1차 구현에서 분산 transaction 문제가 생긴다 | 보류 |
| DB lock 없이 cache decrement | 빠르다 | oversell 0 증명이 어렵다 | 기각 |

## 결과

좋아지는 점:

- 주문 생성과 재고 예약을 같은 DB transaction으로 묶을 수 있다.
- idempotency, reservation TTL, payment saga가 한 aggregate 주변에 모인다.
- oversell 테스트가 명확해진다.

비용:

- `order-service`가 핵심 병목이 될 수 있다.
- admission control과 DB transaction 최적화가 중요해진다.

## 후속 작업

| 상태 | 작업 | 담당 | 연결 문서 |
| --- | --- | --- | --- |
| 계획됨 | inventory bucket과 reservation schema 추가 | 미지정 | ../04-data-design.md |
| 계획됨 | order create transaction 테스트 작성 | 미지정 | ../11-test-release-plan.md |
