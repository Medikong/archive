---
id: SD.A.0120
title: Context 사용자 영속성 설계 인덱스
type: service-design-persistence-index
status: draft
tags: [service-design, user, persistence, postgres, index]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
bounded_context: BC.A.01
domain_model: SD.A.0110
---

# Context 사용자 영속성 설계 인덱스

## 역할

Context 사용자의 PostgreSQL 쓰기 모델, 가입 연동 Inbox/Outbox와 멱등성, 프로필·계정 조회 모델과 인덱스를 안내한다.

## 원천

- [BC.A.01](../../../40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md)
- [SD.A.0110 도메인 모델](../A_01_10-domain-model/README.md)
- [SD.A.30020 인증 영속성](../../A_300_auth/A_300_20-persistence/README.md)
- [SD.A.01 서비스 상세 설계](../README.md)

## 하위 문서

| 문서 | 책임 | 상태 |
| --- | --- | --- |
| [쓰기 모델](write-models.md) | 가입 초안, Provisioning, UserAccount, UserProfile, 상태 이력 스키마와 Repository | draft |
| [신뢰성과 이벤트](reliability-and-events.md) | Inbox/Outbox, 멱등성, 트랜잭션, 재처리와 보상 | draft |
| [조회 모델과 인덱스](read-models-and-indexes.md) | 본인 프로필·마이 조각 조회, 인덱스, 캐시, 보존과 migration | draft |

## 결정 경계

- 사용자 서비스 전용 PostgreSQL이 계정·프로필·Provisioning의 최종 원장이다.
- server와 worker는 DDL을 실행하지 않는다. 별도 migrate Job이 versioned migration을 적용한다.
- Inbox와 업무 변경, Outbox 저장을 같은 로컬 트랜잭션으로 묶는다.
- `user_id`, `auth_registration_id`, `link_request_id`, `profile_request_id` 유일성은 DB constraint로 보장한다.
- private name은 암호화하고 검색용 평문·로그·event 복제본을 만들지 않는다.
- 주문·쿠폰·포인트·등급·찜·알림 요약은 사용자 DB에 저장하지 않는다.
- 닉네임 unique index는 중복 정책 확정 전 만들지 않는다.
