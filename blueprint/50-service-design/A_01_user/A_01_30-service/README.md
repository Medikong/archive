---
id: SD.A.0130
title: Context 사용자 서비스 설계 인덱스
type: service-design-service-index
status: draft
tags: [service-design, user, service, handler, worker, index]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
bounded_context: BC.A.01
domain_model: SD.A.0110
persistence: SD.A.0120
---

# Context 사용자 서비스 설계 인덱스

## 역할

사용자 계정 생성·상태 변경, 프로필 변경, 가입 연동 Process Manager, BFF 사용자 조각 Query와 Event worker를 구현 단위로 안내한다.

## 원천

- [BC.A.01](../../../40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md)
- [SD.A.0110 도메인 모델](../A_01_10-domain-model/README.md)
- [SD.A.0120 영속성](../A_01_20-persistence/README.md)
- [SD.A.30030 인증 서비스](../../A_300_auth/A_300_30-service/README.md)

## 하위 문서

| 문서 | 책임 | 상태 |
| --- | --- | --- |
| [가입·계정 Handler](registration-account-handlers.md) | 가입 초안, Auth Event, 계정 생성·활성·제한·보상 | draft |
| [프로필 Handler](profile-handlers.md) | 본인 프로필, 이미지 upload intent와 자산 연결 | draft |
| [마이 조회 경계](my-query.md) | 사용자 조각 Query와 BFF section 계약 | draft |
| [이벤트 처리](event-processing.md) | Inbox/Outbox, Event Handler, relay, retry, reconciliation | draft |

## 런타임 구성

| 프로세스 | 책임 |
| --- | --- |
| `user-service server` | REST Query/Command, 가입 프로필 초안, 운영 계정 상태 API |
| `user-service worker` | Auth Event consumer, Process Manager, outbox relay, retry·reconciliation |
| `user-service migrate` | versioned PostgreSQL migration 적용 |

## 결정 경계

- server와 worker는 같은 도메인·Repository 코드를 사용하지만 별도 프로세스로 배포한다.
- Handler 하나는 한 업무 Aggregate만 변경한다. Inbox/Outbox/IdempotencyRecord와 Process Manager 조정 행은 그 Aggregate의 트랜잭션에 함께 둘 수 있지만 UserAccount와 UserProfile을 한 트랜잭션에서 함께 변경하지 않는다.
- 변경과 Outbox는 같은 pgx transaction에 저장한다.
- 외부 조회 실패를 성공 기본값으로 바꾸지 않는다.
- 프로필 API는 인증된 본인만, 계정 상태 변경은 workload identity·권한·승인 참조가 있는 운영 경로만 허용한다.
- 마이 대시보드 조합은 BFF가 수행한다. 사용자 서비스는 주문·쿠폰 등 downstream을 호출하지 않는다.
- 현재 공통 JWT 문서와 `X-Principal`은 이행 대상이다. 목표 계약은 `SD.A.30040`을 따른다.

## 코드 구조 후보

```text
services/user-service/
  cmd/server/
  cmd/worker/
  cmd/migrate/
  internal/app/
  internal/domain/registration/
  internal/domain/account/
  internal/domain/profile/
  internal/platform/config/
  internal/platform/observability/
  internal/transport/http/
  internal/transport/kafka/
  api/
  migrations/
```

## 운영 기준

- readiness는 DB schema version, PostgreSQL 연결, 필수 event producer 설정을 확인한다.
- Auth consumer가 지연돼도 기존 active 사용자 API는 제공한다. 가입만 `awaiting_user_link` 상태로 지연된다.
- outbox backlog age, provisioning age, auth link rejection, restriction projection lag를 핵심 metric으로 둔다.
- metric label과 trace attribute에 user ID, private name, event ID를 넣지 않는다.
