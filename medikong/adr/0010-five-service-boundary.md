---
id: ADR-0010
title: "DropMong을 5개 서비스로 시작한다"
status: proposed
date: 2026-07-02
areas:
  - architecture
  - services
repos:
  - workspaces
  - services
decision_drivers:
  - oversell 방지
  - 구현 집중도
  - 데모 명확성
related:
  - ../03-service-boundaries.md
links: []
supersedes: []
superseded_by: null
---

# ADR 0010: DropMong을 5개 서비스로 시작한다

## 상태

Proposed

## 배경

기존 Ticketmong 구조는 `auth`, `concert`, `reservation`, `payment`, `ticket`, `notification` 중심이다. DropMong은 제한 수량 커머스이므로 `concert`, `reservation`, `ticket` 경계를 그대로 유지하면 제품 도메인과 맞지 않고, oversell 0 검증도 흩어진다.

## 결정

DropMong은 1차 구현에서 다음 5개 서비스로 시작한다.

- `auth-service`
- `catalog-service`
- `order-service`
- `payment-service`
- `notification-service`

`concert-service`는 `catalog-service`로 전환하고, `reservation-service`와 `ticket-service`의 핵심 책임은 `order-service`로 합친다.

## 대안

| 대안 | 장점 | 단점 | 판단 |
| --- | --- | --- | --- |
| 기존 6개 서비스 유지 | 변경량이 적어 보인다 | DropMong 도메인과 맞지 않고 ticket 책임이 애매하다 | 기각 |
| inventory-service 추가 | 재고 책임이 명시적이다 | 1차 구현에서 transaction 경계가 분산된다 | 보류 |
| waiting-room-service 추가 | 피크 트래픽 제어가 명확하다 | MVP 범위를 키운다 | 보류 |

## 결과

좋아지는 점:

- 발표와 구현 범위가 단순하다.
- order와 inventory 불변 조건을 한 트랜잭션으로 검증할 수 있다.
- 기존 Ticketmong 흔적을 DropMong 도메인으로 명확히 바꿀 수 있다.

비용:

- 기존 reservation/ticket 코드는 재구성이 필요하다.
- 나중에 inventory 또는 waiting room 분리가 필요할 수 있다.

## 후속 작업

| 상태 | 작업 | 담당 | 연결 문서 |
| --- | --- | --- | --- |
| 계획됨 | `services` 서비스 디렉터리와 계약 rename 계획 | 미지정 | ../03-service-boundaries.md |
| 계획됨 | `e-gitops` 서비스 values rename 계획 | 미지정 | ../08-infra-deployment.md |
