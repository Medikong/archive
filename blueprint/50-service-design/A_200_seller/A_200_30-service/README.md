---
id: SD.A.20030
title: Context 판매자 서비스 설계 인덱스
type: service-design-service-index
status: draft
tags: [service-design, seller, application-service, msa]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
---

# Context 판매자 서비스 설계 인덱스

이 영역은 `seller-service`라는 단일 배포 단위를 설계하지 않는다. `CMD.A.200-01~17`과 `RM.A.200-01~10`을 기존 MSA에 배치할 때 필요한 Handler, Event, 보안 계약을 정의하며, 실제 배치는 [MSA 서비스 구성도](../README.md#실제-msa-서비스-구성도)를 따른다.

| 책임 | 목표 배포 단위 | 상태 |
| --- | --- | --- |
| Seller Management | 소유 서비스 미확정 | Auth에 넣지 않으며 구현 대기 |
| Seller Proposal | `catalog-service` 배정 후보 | seller Aggregate·API 미구현 |
| Seller Operations Query | 소유 서비스 미확정 | 교차 서비스 투영 배포 금지 |
| Order Export | `order-service` 배정 후보 | seller 조회·감사·파일 계약 미구현 |
| Seller Issue | 소유 서비스 미확정 | 플랫폼 운영 case 계약 대기 |

| 문서 | 내용 |
| --- | --- |
| [Handler와 책임 경계](handlers-and-boundaries.md) | `CMD.A.200-01~17`, `RM.A.200-01~10`, 목표 서비스와 트랜잭션 Port |
| [Event 처리](event-processing.md) | 서비스별 outbox/inbox, 원천 Event와 계약 대기 상태 |
| [보안·오류·가용성](security-errors-and-availability.md) | Ingress 신뢰 경계, membership, 강한 재인증, 동일 404, 403/409/503, stale·partial |

Controller는 인증 context와 wire 입력을 검증하고 해당 서비스의 Handler를 호출한다. 한 HTTP Command는 한 서비스의 한 Aggregate 트랜잭션만 직접 실행하며, 다른 서비스의 후속 작업은 소유 서비스가 발행한 versioned Event 또는 명시적 API 계약으로 처리한다. 소유 서비스가 미정인 Handler 이름은 논리 설계이며 배포 가능한 구현 계약이 아니다.
