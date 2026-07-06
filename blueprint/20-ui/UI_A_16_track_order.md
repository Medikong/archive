---
id: UI.A.16
title: 배송 조회 페이지 UI
type: ui-asset
status: draft
tags: [ui, screenshot, delivery-tracking, shipment, component-sheet, dropmong, limited-commerce]
source: local
created: 2026-07-07
updated: 2026-07-07
---

# 배송 조회 페이지 UI

## 기본 정보

- UI ID: `UI.A.16`
- 연관 Page: [PAGE.A.16](../10-sitemap/PAGE_A_16_track_order.md)
- 에셋 유형: 화면 이미지, 컴포넌트 시트
- 파일 경로:
  - [배송 조회 페이지](assets/UI_A_16_track_order/UI_A_16_01_track_order.png)
  - [배송 조회 페이지 컴포넌트 시트](assets/UI_A_16_track_order/UI_A_16_02_track_order_component.png)
- 원본 URL: local
- 캡처 일시: 2026-07-07
- 캡처 조건: DropMong 배송 조회, 배송중 상태, 배송 진행 스텝, 배송 타임라인, 배송지 정보, 하단 액션 버튼 상태

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.01](../00-requirements/REQ_A_01_limited_drop_commerce.md) | 페이지 참조: [PAGE.A.16](../10-sitemap/PAGE_A_16_track_order.md) | UC 참조: UC.A.16 | 영속성 참조: PST.A.16 | 서비스 참조: SVC.A.16 | 시나리오 참조: SCN.A.16 | API 참조: API.A.16

## 에셋

### 배송 조회 페이지

![배송 조회 페이지](assets/UI_A_16_track_order/UI_A_16_01_track_order.png)

### 컴포넌트 시트

![배송 조회 페이지 컴포넌트 시트](assets/UI_A_16_track_order/UI_A_16_02_track_order_component.png)

## 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상단 앱 바 | 뒤로가기, 페이지 제목, 알림 진입을 제공한다. | 뒤로가기, 알림 |
| 2 | 배송 요약 카드 | 대표 상품 이미지와 주문번호, 택배사, 운송장 번호, 예상 도착일을 보여준다. | 택배사 조회 |
| 3 | 배송 상태 헤더 | 현재 배송 상태와 보조 메시지를 크게 보여준다. | 현재 상태 확인 |
| 4 | 배송 진행 스텝 | 주문완료, 배송준비, 배송중, 배송완료 단계를 가로 스텝으로 보여준다. | 완료/현재/미완료 상태 |
| 5 | 배송 타임라인 | 배송 이벤트 이력을 시간순으로 보여준다. | 현재 이벤트 강조 |
| 6 | 배송지 정보 카드 | 수령인, 연락처, 배송 주소를 보여준다. | 배송지 확인 |
| 7 | 하단 액션 버튼 | 택배사 조회, 배송 기사 문의, 주문 상세 보기 버튼을 제공한다. | 외부 조회, 전화, 주문 상세 |
| 8 | 상태 칩/배지 | 주문완료, 배송준비, 배송중, 배송완료, 도착 예정 상태 칩을 정의한다. | 상태 표시 |
| 9 | 아이콘 모음 | 뒤로가기, 알림, 위치, 배송, 박스, 전화, 캘린더, 완료, 비활성 아이콘을 정의한다. | 아이콘 상태 |
| 10 | 상품 썸네일 예시 | 배송 조회에 표시될 대표 상품 이미지 예시를 제공한다. | 상품 이미지 표시 |
| 11 | 마스코트/일러스트 | 배송 준비 중, 배송 중, 배송 완료, 문제 발생 상태 일러스트를 정의한다. | 배송 상태 시각화 |
| 12 | 하단 내비게이션 바 | 홈, 드롭, 알림, 마이 탭을 제공한다. | 전역 탭 이동 |

## 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 배송 | `trackingId` | string | 배송 조회 식별 |
| 주문 | `orderId` | string | 주문 식별 |
| 주문 | `orderNumber` | string | 주문번호 표시 |
| 상품 | `representativeProduct.productId` | string | 상품 상세 연결 |
| 상품 | `representativeProduct.thumbnailUrl` | image | 대표 상품 썸네일 표시 |
| 택배 | `carrier.name` | string | 택배사명 표시 |
| 택배 | `carrier.code` | string | 택배사 연동 코드 |
| 택배 | `invoiceNumber` | string? | 운송장 번호 표시 |
| 배송 | `expectedArrivalDate` | date? | 예상 도착일 표시 |
| 배송 | `currentStatus` | enum | 주문완료, 배송준비, 배송중, 배송완료 |
| 배송 | `currentStatusLabel` | string | 현재 상태 제목 표시 |
| 배송 | `currentStatusMessage` | string | 현재 상태 설명 표시 |
| 진행 스텝 | `steps[].status` | enum | 단계 식별 |
| 진행 스텝 | `steps[].label` | string | 단계 라벨 표시 |
| 진행 스텝 | `steps[].completedAt` | datetime? | 단계 완료 시각 표시 |
| 진행 스텝 | `steps[].state` | enum | 완료, 현재, 미완료 |
| 타임라인 | `events[].occurredAt` | datetime? | 이벤트 시각 표시 |
| 타임라인 | `events[].title` | string | 이벤트 제목 표시 |
| 타임라인 | `events[].description` | string | 이벤트 설명 표시 |
| 타임라인 | `events[].current` | boolean | 현재 이벤트 강조 |
| 배송지 | `shippingAddress.recipientName` | string | 수령인 표시 |
| 배송지 | `shippingAddress.phone` | string | 연락처 표시 |
| 배송지 | `shippingAddress.addressLine` | string | 주소 표시 |
| 액션 | `actions.canOpenCarrierTracking` | boolean | 택배사 조회 활성 |
| 액션 | `actions.canCallDriver` | boolean | 배송 기사 문의 활성 |
| 액션 | `actions.canViewOrderDetail` | boolean | 주문 상세 보기 활성 |

## 화면에서 확인한 행동

- 사용자는 현재 배송 상태가 배송중임을 확인한다.
- 사용자는 주문번호, 택배사, 운송장 번호, 예상 도착일을 확인한다.
- 사용자는 주문완료, 배송준비, 배송중, 배송완료 단계 중 현재 위치를 확인한다.
- 사용자는 배송 타임라인에서 상세 이벤트와 현재 이벤트를 확인한다.
- 사용자는 배송지 정보를 확인한다.
- 사용자는 택배사 조회, 배송 기사 문의, 주문 상세 보기로 이동한다.

## 설계 반영 사항

- Read Model 후보: `RM.A.16 DeliveryTrackingReadModel`
- Query 후보: `QRY.A.17.GetDeliveryTracking`, `QRY.A.18.GetDeliveryTimeline`
- Command 후보: `CMD.A.30.OpenCarrierTracking`, `CMD.A.31.CallDeliveryDriver`, `CMD.A.32.ViewOrderDetailFromTracking`
- Event 후보: `EVT.A.17.DeliveryPrepared`, `EVT.A.18.DeliveryPickedUp`, `EVT.A.19.DeliveryInTransit`, `EVT.A.20.DeliveryCompleted`
- Error 후보: `ERR.A.30.TRACKING_NOT_FOUND`, `ERR.A.31.INVOICE_NOT_READY`, `ERR.A.32.CARRIER_TRACKING_UNAVAILABLE`, `ERR.A.33.TRACKING_ACCESS_DENIED`
- 권한 후보: 배송 조회는 주문 소유자 로그인 필요

## 확인 필요

- 택배사별 운송장 조회 URL과 앱 내부/외부 브라우저 정책
- 배송 기사 문의 전화번호 제공 기준
- 배송 타임라인 이벤트 정규화 여부
- 예상 도착일과 현재 상태 업데이트 주기
- 배송완료 이후 구매확정 CTA를 배송 조회 화면에 추가할지, 배송/주문 관리 화면으로 보낼지 여부
- 배송 이력 없음, 운송장 없음, 택배사 장애 상태의 빈/오류 화면
