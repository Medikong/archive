---
id: SD.A.20030.EVENTS
title: 판매자 Event 발행·소비 서비스 설계
type: service-design-service
status: draft
tags: [service-design, seller, event, outbox, inbox]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
service: SD.A.20030
---

# 판매자 Event 발행·소비 서비스 설계

Event relay와 consumer는 별도 Seller 런타임에 모이지 않는다. Catalog 소유 Aggregate의 outbox는 `catalog-service`, OrderExport outbox는 `order-service`가 처리한다. 소유 서비스가 미확정인 Event와 projection은 논리 계약 상태로만 유지한다.

## 발행

- 각 소유 서비스의 outbox relay는 commit된 seller Aggregate Event만 읽고 같은 `event_id`로 최소 한 번 발행한다.
- relay 재시도, broker ack와 publish attempt는 업무 Aggregate version을 변경하지 않는다.
- `EVT.A.200-01~17`의 envelope와 최소 payload는 [Event 계약](../A_200_40-api/event-contracts.md)을 따른다.
- broker·topic·partition·보존 기간은 BC·REQ가 정하지 않았으므로 배포 계약으로 확정하지 않는다.

## 소비와 투영

| 원천 | 필요한 사실 | 대상 Read Model | 현재 상태 |
| --- | --- | --- | --- |
| Drop/Catalog | 승인 snapshot 수신, 게시·오픈·종료·차단, seller/drop/product/option ref | `RM.A.200-05~06`, `RM.A.200-08` | 외부 계약 대기 |
| Order | 주문 생성·상태·취소, seller 귀속 item, 최소 출고 snapshot | `RM.A.200-05`, `RM.A.200-07~08` | 외부 계약 대기 |
| Payment | 승인·실패·취소, seller 귀속 금액과 order ref | `RM.A.200-05`, `RM.A.200-08` | 외부 계약 대기 |
| Coupon | seller campaign·발급·사용·회수·비용 귀속 | `RM.A.200-05`, `RM.A.200-08~09` | 설계 보강·구현 대기 |
| Settlement | 예정·보류·확정·차감·지급 snapshot | `RM.A.200-09` | 외부 계약 대기 |
| 플랫폼 검수·운영 | review decision, issue case result | `RM.A.200-04`, `RM.A.200-10` | 외부 계약 대기 |

현재 서비스 코드의 `OrderCreatedEvent`, `PaymentApprovedEvent`, `PaymentFailedEvent`, `NotificationRequestedEvent`는 seller 귀속·version·취소·정산에 필요한 계약을 충분히 제공한다는 근거가 없다. 이름이 비슷하다는 이유로 바로 소비하지 않고 소유 Context의 versioned schema가 확정될 때 adapter를 만든다.

`RM.A.200-05`, `RM.A.200-08~10`의 consumer 배포 위치는 미정이다. 브라우저, Ingress, `dropmong-web` 또는 임의의 기존 서비스가 임시 consumer가 되어 집계하지 않는다.

## 실패 처리

- schema 미지원, payload hash 충돌, version gap은 성공으로 ack하지 않고 failure ledger와 DLQ ref를 남긴다.
- 업무상 무관한 seller/item Event는 정상 무시 결과로 기록한다.
- 재처리는 같은 event ID를 사용하며 metric bucket contribution을 중복시키지 않는다.
- 개인정보가 포함된 원본 payload를 실패 로그·DLQ 운영 화면에 복제하지 않는다.
