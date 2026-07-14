---
id: SD.A.0120
title: Context 사용자 영속성 설계
type: service-design-persistence
status: draft
tags: [service-design, user, persistence, postgresql]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 영속성 설계

## 저장 모델

| 테이블 | 역할 |
| --- | --- |
| `users` | 계정 상태와 프로필의 단일 쓰기 원장 |
| `user_agreement_acceptances` | 가입 시점의 필수 동의 이력 |
| `user_status_history` | 계정 상태 변경 감사 이력 |
| `user_idempotency_records` | 생성·변경 API의 멱등 결과 |

가입 초안, Provisioning, Inbox, Outbox와 Media upload intent를 사용자 DB에 저장하지 않는다.

## 문서

- [쓰기 모델](write-models.md)
- [조회 모델과 인덱스](read-models-and-indexes.md)
- [멱등성과 실패 처리](reliability-and-events.md)

## 트랜잭션

| 작업 | 하나의 PostgreSQL 트랜잭션 |
| --- | --- |
| User 생성 | `users` + 필수 동의 이력 + 멱등 결과 |
| 프로필 수정 | 조건부 `users` UPDATE + 멱등 결과 |
| 이미지 연결 | 조건부 `users` UPDATE + 멱등 결과 |
| 계정 상태 변경 | 조건부 `users` UPDATE + 상태 이력 + 멱등 결과 |

외부 서비스 호출은 User 트랜잭션에 포함하지 않는다. 프론트엔드는 Ingress를 통해 Auth와 Media를 별도로 호출하며 각 서비스 계약에 따라 재시도한다.
