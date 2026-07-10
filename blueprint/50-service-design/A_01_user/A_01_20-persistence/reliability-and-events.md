---
id: SD.A.0120.RELIABILITY
title: Context 사용자 신뢰성과 이벤트 영속성
type: service-design-persistence
status: draft
tags: [service-design, user, inbox, outbox, idempotency, transaction]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 신뢰성과 이벤트 영속성

## 저장 모델

| 테이블 | 역할 | 핵심 유일성 |
| --- | --- | --- |
| `user_inbox_events` | 외부 Event 수신, 검증, 재시도와 결과 | `(consumer_name, source_event_id)` |
| `user_outbox_events` | 업무 Event와 integration Event 전달 | `event_id` |
| `user_idempotency_records` | HTTP·운영 Command 재시도 결과 | `(operation_namespace, scope_hash, idempotency_key_digest)` |

## Inbox

| 필드 | 설명 |
| --- | --- |
| `consumer_name`, `source_event_id` | transport 중복 제거 키 |
| `event_type`, `event_version`, `producer` | allowlist와 version 검증 |
| `aggregate_id`, `correlation_id`, `causation_id` | 업무 상관관계 |
| `payload_hash` | 같은 event ID의 다른 payload 탐지 |
| `received_at` | 사용자 consumer 접수 시각. 로컬 처리 가능성 판단용이며 Auth 수락 시각이 아님 |
| `status` | received/processing/deferred/processed/rejected/dead |
| `attempt_count`, `next_attempt_at`, `last_error_code` | 재시도 상태 |
| `lease_owner`, `lease_until` | processing worker 소유권과 장애 복구 기한 |
| `result_ref` | Provisioning·UserAccount 등 처리 결과 |

같은 event ID와 같은 payload는 기존 결과를 반환한다. 다른 payload는 보안 충돌로 `rejected` 처리하고 업무 상태를 바꾸지 않는다.

Worker는 ready row를 `FOR UPDATE SKIP LOCKED`로 가져와 짧은 트랜잭션 안에서 `processing`과 lease를 설정한다. 처리 중 죽으면 `lease_until`이 지난 row를 `deferred`로 되돌려 재시도하므로 영구 정체시키지 않는다. 외부 Port 호출 동안 row lock은 유지하지 않는다.

## Outbox

| 필드 | 설명 |
| --- | --- |
| `event_id`, `event_type`, `event_version` | 계약 식별자 |
| `aggregate_type`, `aggregate_id`, `aggregate_version` | 발생 원장 |
| `correlation_id`, `causation_id` | 인과관계 |
| `payload` | 민감 원문을 제외한 JSONB |
| `status` | pending/publishing/published/dead |
| `attempt_count`, `next_attempt_at`, `lease_until` | relay 상태 |
| `created_at`, `published_at` | 관측 시각 |

Outbox는 Aggregate 변경과 같은 PostgreSQL transaction에 저장한다. broker publish 성공을 업무 Event 이름으로 사용하지 않는다.

## HTTP 멱등성

- 생성·변경 API는 `Idempotency-Key`를 요구한다.
- 유일 범위는 `API ID/operation + actor 또는 사전 인증 context의 scope_hash + key digest`다.
- 본인 API scope는 검증된 `user_id`, 가입 초안은 BFF workload subject + `registration_process_id`, 운영 변경은 operator/workload + target user ID를 사용한다.
- key 원문 대신 keyed digest, scope HMAC, command type, canonical request HMAC fingerprint, 결과 참조와 TTL을 저장한다.
- 같은 key와 같은 payload는 기존 결과를 반환한다.
- 같은 key와 다른 payload는 `409 USER_IDEMPOTENCY_CONFLICT`다.
- 프로필 version 충돌과 멱등성 충돌을 다른 오류 코드로 구분한다.
- 기존 결과를 재생하기 전에 현재 workload/Principal, 리소스 소유권, 계정 상태와 운영 승인을 다시 확인한다. 다른 actor에게 저장 결과를 반환하지 않는다.

`user_idempotency_records`는 최소한 다음 값을 가진다.

| 필드 | 역할 |
| --- | --- |
| `idempotency_record_id` | 레코드 전역 ID |
| `operation_namespace`, `scope_hash`, `idempotency_key_digest` | 유일 업무 범위와 HMAC 기반 key 식별 |
| `request_fingerprint` | canonical 요청 전체의 keyed HMAC; 원문과 단순 hash는 저장하지 않음 |
| `external_operation_id` | 미디어처럼 외부 멱등 호출에 전달할 안정적인 operation ID |
| `status` | processing/completed/failed |
| `resource_type`, `resource_id`, `resource_version`, `result_code` | 비민감 결과 재구성 참조 |
| `result_metadata` | 상태, 변경 필드, 완료 시각처럼 allowlist로 제한한 비민감 JSON |
| `created_at`, `completed_at`, `expires_at` | 선점·완료·보존 기한 |

UNIQUE `(operation_namespace, scope_hash, idempotency_key_digest)`를 둔다. response body, private name, signed/upload URL은 저장하지 않고 Aggregate ID, version, allowlist 결과 metadata만 재구성한다.

## 트랜잭션 경계

| 작업 | 같은 트랜잭션 | 후속 작업 |
| --- | --- | --- |
| 가입 초안 생성 | draft + idempotency record | 없음 |
| 이미지 upload intent 선점 | processing idempotency + 고정 external operation ID | 트랜잭션 밖에서 Media CreateOrGet 호출 |
| 이미지 upload intent 확정 | 같은 idempotency record + media intent/asset ref + expiresAt | upload URL은 Media에서 응답하고 저장하지 않음 |
| Auth 검증 Event 접수·업무 선점 | inbox received/processing + 최소 Provisioning/link request ID + agreement pending | 트랜잭션 밖에서 Agreement Port 검증 |
| Auth 검증 Event 승인 처리 | 기존 Provisioning agreement valid + draft consume + 고정 account/profile Command ID + account Command outbox | UserAccount Handler 실행 |
| Auth 검증 Event 업무 거부 | inbox processed + 기존 Provisioning agreement invalid/rejected + 존재하는 draft rejected/귀속 + `User.AuthLinkRejected` outbox | Auth가 Registration 실패 반영 |
| UserAccount 생성 | account + domain outbox | Process Manager가 account flag + 고정 profile Command outbox 저장 |
| UserProfile 생성 | profile + domain outbox | Process Manager의 profile 완료 조건 반영 |
| Auth link 요청 | Provisioning requested + integration outbox | relay가 broker 발행 |
| Auth link 성공 | inbox + account active + Provisioning linked + outbox | 일반 조회 허용 |
| Auth link 거부/timeout | inbox + Provisioning terminal + 존재하는 draft rejected/expired와 귀속 + 존재하는 account만 provisioning_failed + outbox | 초안 또는 생성된 개인정보 정리 Worker 예약 |
| 프로필 수정 | account 공유 잠금 + active 재확인 + profile + idempotency + outbox | 이전 media asset 정리 요청 |
| 계정 상태 변경 | account + status history + idempotency + auth-state outbox | Auth projection 반영 |

Process Manager가 여러 Aggregate를 하나의 transaction으로 강제로 묶지 않는다. 각 Aggregate 결과 Event를 Inbox/Outbox로 연결하고 현재 상태를 재확인한다.

프로필 정보·이미지 변경은 같은 DB에서 UserAccount를 `FOR SHARE`, UserProfile을 `FOR UPDATE` 순서로 잠근다. 계정 상태 변경은 UserAccount를 `FOR UPDATE`로 잠그므로 두 작업이 직렬화된다. 모든 Handler가 account -> profile 순서를 지켜 deadlock을 피하고, 잠금을 얻은 뒤 active와 expected profile version을 다시 확인한다.

이미지 upload intent는 외부 호출을 DB 트랜잭션에 넣지 않는다. 첫 트랜잭션이 actor scope의 IdempotencyRecord와 `external_operation_id`를 선점하고, Media Port에는 이 ID를 idempotency key로 전달한다. 다음 트랜잭션이 같은 record에 media intent/asset 참조와 만료 시각을 확정한다. 외부 성공 뒤 worker가 죽어도 재시도는 같은 external operation ID로 `CreateOrGet`을 호출하므로 asset을 중복 생성하지 않는다.

검증 Event를 수락하는 첫 트랜잭션은 Inbox와 최소 Provisioning, `link_request_id`, `agreement_validation_status=pending`을 함께 저장한다. 이 중 하나만 커밋하지 않는다. 그 뒤 DB 트랜잭션 밖에서 Agreement Port를 호출한다. 승인 처리 트랜잭션은 Provisioning, Inbox와 초안을 다시 잠근 뒤 terminal 여부, receipt 결과와 초안 상태를 확인하고 초안 소비, 고정 `account_command_id`·`profile_command_id`와 Account Command Outbox를 함께 저장한다. `UserAccountCreated`를 반영하는 트랜잭션이 account 완료 flag와 Profile Command Outbox를 함께 저장하고, `DefaultProfileCreated` 반영 뒤에만 link 요청을 만든다. Profile Command를 두 경로에서 예약하지 않는다.

Agreement 호출 중 `Auth.RegistrationUserLinkRejected(reason=LINK_TIMEOUT)`이 도착하면 registration/correlation으로 이미 저장된 nonterminal Provisioning을 잠근다. Provisioning을 `timed_out`, 존재하는 draft를 `expired`와 Provisioning 귀속으로 바꾸며 정리 시각을 기록한다. 늦게 돌아온 Agreement 결과와 Account/Profile Command Handler는 terminal PM을 다시 확인하고 계정·프로필 생성을 시작하지 않는다.

가입 생성·결과 Handler의 잠금 순서는 Provisioning -> UserAccount -> UserProfile이다. Account/Profile 생성 트랜잭션은 Provisioning을 `FOR SHARE`로 먼저 잠근 채 nonterminal을 다시 확인하고 Aggregate와 Outbox를 커밋한다. timeout Handler는 같은 Provisioning을 `FOR UPDATE`로 잠근 뒤 terminal로 닫고, 생성 Handler가 먼저 커밋한 account가 있으면 이어서 `provisioning_failed`와 프로필 정리 대상을 기록한다. 따라서 terminal 확인과 Aggregate insert 사이에 timeout이 끼어들 수 없다.

partition 재배치나 consumer 간 지연으로 timeout 결과가 원 검증 Event보다 먼저 보이면 결과 Inbox를 `deferred`로 두고 원 Event/Provisioning을 기다린다. 제한된 순서 역전 grace 안에 원 Event가 도착하지 않을 때만 계약 위반으로 격리하며, 정보가 부족한 상태에서 임시 Provisioning을 합성하지 않는다.

## 가입 업무 멱등성

- `source_event_id`: transport 중복 제거
- `auth_registration_id`: Auth Registration당 사용자 한 명
- `verification_binding_id`: 검증 snapshot당 가입 한 건
- `profile_request_id`: 프로필 초안 1회 소비
- `link_request_id`: Auth 계정 연동 업무 한 건
- `account_command_id`, `profile_command_id`: Aggregate 생성 Command 재전달 멱등 키
- `user_id`: 사용자 계정 전역 유일성

어느 유일성 충돌도 기존 `user_id`를 외부 오류에 노출하지 않는다. 같은 업무이면 기존 결과를 이어가고, 다른 binding/payload이면 보안 충돌로 격리한다.

## 재시도와 보상

- DB·broker·동의 검증 timeout은 Inbox를 `deferred`로 두고 backoff 재시도한다.
- 프로필·동의가 업무 규칙상 유효하지 않을 때만 `User.AuthLinkRejected`를 발행한다.
- 사용자 서비스는 `link_accept_until`과 clock skew·broker transport budget을 보고 계정 생성 시작 가능성을 fail closed로 판단하고, link outbox 저장·발행 전 남은 시간을 다시 확인한다. 최종 수락 여부는 `User.AuthLinkRequested`가 Auth Inbox에 도착한 시각으로 Auth가 결정하며 로컬 grace는 이 기한을 연장하지 않는다.
- Auth link 거부·기한 만료 뒤 UserAccount/UserProfile을 즉시 삭제하지 않는다. `provisioning_failed`로 닫고 보존 정책 뒤 crypto-shred·정리를 수행한다.
- outbox dead 상태는 자동 성공 처리하지 않고 온콜 경보와 수동 재처리 대상으로 남긴다.
- Auth link 전달 재시도는 기존 Outbox row와 `link_event_id`를 다시 queue한다. 같은 `link_request_id`에 새 event ID를 만들어 결과 `causation_id` 후보를 늘리지 않는다.

## Relay와 Worker

- Inbox와 Outbox는 `FOR UPDATE SKIP LOCKED`로 대상을 선점하고 DB에 소유자·만료 시각을 기록하는 lease를 사용한다.
- 한 레코드 처리 성공 뒤에만 broker offset 또는 outbox published 상태를 확정한다.
- lease는 publish timeout보다 길어야 하고, 종료 시 신규 lease를 중단한 뒤 진행 중 작업을 drain한다.
- 재시도 label에 user ID, event ID 같은 고카디널리티 값을 넣지 않는다.
- processing Inbox 또는 `link_status=not_requested` Provisioning에서 account가 없으면 같은 Account Command ID를, account만 준비됐으면 같은 Profile Command ID를 다시 저장한다. 새 ID를 만들거나 Profile Command를 두 단계에서 중복 예약하지 않는다.

## 확인 필요

- inbox/outbox 보존 기간과 dead-letter 운영 승인 절차
- `link_accept_until` 처리 grace와 clock skew 허용치
- Provisioning 정리 Worker의 보존·crypto-shred 정책
- 계정 제한 Event 소비 지연 시 Session 폐기 SLO와 수동 재전달 승인 절차
