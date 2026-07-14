---
id: SD.A.20010.READ_MODELS
title: 판매자 Read Model 소유권과 최신성
type: service-design-read-model
status: draft
tags: [service-design, seller, read-model, freshness]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
domain_model: SD.A.20010
---

# 판매자 Read Model 소유권과 최신성

| Read Model | 목표 배포 단위 | 배포 상태 |
| --- | --- | --- |
| `RM.A.200-01~02` | 미확정 | 제공 불가 |
| `RM.A.200-03~04`, `RM.A.200-06` | `catalog-service` 후보 | seller 계약 미구현 |
| `RM.A.200-07` | `order-service` 후보 | seller 주문·export 계약 미구현 |
| `RM.A.200-05`, `RM.A.200-08~10` | 미확정 | 교차 서비스 조회·정산·issue 소유자 필요 |

| Read Model | 제공 내용 | 원천 | 실패 표현 |
| --- | --- | --- | --- |
| `RM.A.200-01` | 판매자·스토어 설정, 책임 고지 미리보기 | SellerAccount·StoreProfile 원장 | 필수 원장 실패는 typed `503` |
| `RM.A.200-02` | membership·role·permission·변경 이력 | SellerMembership·SellerRole·감사 | stale scope는 `409`, 저장소 실패는 `503` |
| `RM.A.200-03` | seller 상품·option·공급 선언 | SellerProduct 원장 | 다른 seller와 없음은 같은 `404` |
| `RM.A.200-04` | proposal·review·변경 이력 | DropProposal 원장, 검수 결과 ref | 검수 원천 지연 section은 stale/partial |
| `RM.A.200-05` | 우선 작업·KPI·진행 drop·최근 주문·일정 | Seller 원장과 외부 Event 투영 | section별 `unavailableSections` |
| `RM.A.200-06` | proposal과 공개 drop 상태 대조 | Seller proposal + Drop/Catalog Event | 공개 원천 미계약은 unavailable |
| `RM.A.200-07` | seller 주문·출고 최소 정보·export 상태 | Order Event/snapshot + OrderExport | 개인정보 원천 장애는 빈 목록이 아닌 typed `503` 또는 partial |
| `RM.A.200-08` | 상품·option·drop·유입·쿠폰·취소 지표 | Drop/Order/Payment/Coupon Event | `asOf`, `sourceWatermark`, `stale`, 추정/확정 구분 |
| `RM.A.200-09` | 정산 예정·보류·확정·차감 snapshot | Settlement Event/API, Coupon cost ref | 계약 전 section unavailable |
| `RM.A.200-10` | issue·추가 정보·외부 결과 timeline | SellerIssue + 운영 case Event | 외부 결과 지연을 현재 SellerIssue 상태와 구분 |

Query가 구현되면 전체 `asOf`와 section별 `sourceAsOf`, `sourceWatermark`, `stale`, `partial`, `unavailableReasonCode`를 제공한다. 소유 서비스가 미정인 Query에는 성공 응답을 가정하지 않는다. 지표별 허용 지연 수치는 소유 Context의 최신성 계약이 정해지기 전까지 운영 기본값으로 확정하지 않는다.
