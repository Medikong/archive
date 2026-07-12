---
id: SD.A.0120.WRITE-MODELS
title: Context 사용자 쓰기 모델
type: service-design-persistence
status: draft
tags: [service-design, user, postgres, schema, repository]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 쓰기 모델

## 저장 모델

| 테이블 | 역할 | 도메인 모델 |
| --- | --- | --- |
| `user_registration_drafts` | 가입 전 프로필 입력과 `profile_request_id` | UserRegistrationDraft |
| `user_provisionings` | Auth 검증부터 연동 결과까지 장기 작업 | UserProvisioning |
| `user_accounts` | `user_id`, 계정·Auth 상태와 version | UserAccount |
| `user_profiles` | 이름·닉네임·인사말·미디어 자산 참조 | UserProfile |
| `user_account_status_history` | 계정 상태 변경 append-only 이력 | AccountStatusChange |

모든 시각은 `TIMESTAMPTZ`, 업무 version은 `BIGINT`, ID는 API에서 string으로 직렬화한다. `user_id`는 PostgreSQL `UUID`를 사용한다.

## user_registration_drafts

| 필드 | 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `profile_request_id` | VARCHAR(128) | PK | 외부에 전달할 opaque ID |
| `registration_process_id` | VARCHAR(128) | NOT NULL, UNIQUE | BFF 가입 작업 scope |
| `private_name_ciphertext` | BYTEA | NULL | pending/프로필 준비 전에는 필수, terminal 정리 뒤 NULL |
| `private_name_key_version` | VARCHAR(64) | NULL | ciphertext와 함께 crypto-shred |
| `nickname_candidate` | VARCHAR(255) | NULL | 정책 확정 전 DB 길이는 상한만 방어 |
| `referral_attribution_id` | VARCHAR(128) | NULL | 프로모션 원천 참조 |
| `status` | VARCHAR(32) | NOT NULL, CHECK | pending/consumed/expired/rejected |
| `expires_at` | TIMESTAMPTZ | NOT NULL | 가입 재사용 기한 |
| `consumed_by_provisioning_id` | VARCHAR(128) | NULL, UNIQUE, FK user_provisionings | consumed/rejected/expired로 닫은 Provisioning 귀속 |
| `version` | BIGINT | NOT NULL, CHECK `> 0` | 낙관적 version |
| `created_at`, `updated_at` | TIMESTAMPTZ | NOT NULL | 서버 시각 |

`private_name_ciphertext`의 deterministic hash나 검색용 평문 열은 만들지 않는다. 이름 검색이 필요한 운영 요구가 생기면 별도 승인된 검색 모델을 설계한다.

`status='pending'`이면 ciphertext와 key version이 모두 NOT NULL이어야 한다. 성공적으로 소비한 초안은 UserProfile 생성과 Auth 연동 안정화 뒤 두 값을 NULL로 만들 수 있으며 멱등 tombstone은 유지한다.

```sql
CHECK (status IN ('pending', 'consumed', 'expired', 'rejected')),
CHECK (
  (private_name_ciphertext IS NULL) = (private_name_key_version IS NULL)
),
CHECK (
  status <> 'pending' OR private_name_ciphertext IS NOT NULL
)
```

## user_provisionings

| 필드 | 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `provisioning_id` | VARCHAR(128) | PK | 가입 작업 식별자 |
| `source_event_id` | VARCHAR(128) | NOT NULL, UNIQUE | Auth event 중복 방지 |
| `auth_registration_id` | VARCHAR(128) | NOT NULL, UNIQUE | Auth Registration 1:1 |
| `registration_version` | BIGINT | NOT NULL | Auth binding snapshot version |
| `verification_binding_id` | VARCHAR(128) | NOT NULL, UNIQUE | 검증 binding |
| `verification_snapshot_hash` | VARCHAR(256) | NOT NULL | 수신 hash 원문 보존 |
| `profile_request_id` | VARCHAR(128) | NOT NULL, UNIQUE | 이벤트가 가리킨 초안 참조. 존재하지 않는 초안의 거부 결과도 보존하므로 FK를 두지 않는다. |
| `agreement_receipt_id` | VARCHAR(128) | NOT NULL, UNIQUE | 가입 작업·프로필 초안에 묶인 1회성 외부 동의 receipt |
| `agreement_validation_status` | VARCHAR(16) | NOT NULL, CHECK | pending/valid/invalid |
| `agreement_binding_hash` | VARCHAR(256) | NULL | valid 응답의 귀속·약관 version·유효 시각 hash |
| `source_occurred_at` | TIMESTAMPTZ | NOT NULL | producer event 시각 |
| `received_at` | TIMESTAMPTZ | NOT NULL | 사용자 consumer 접수 시각; Auth 최종 수락 판정 기준이 아님 |
| `link_accept_until` | TIMESTAMPTZ | NOT NULL | Auth 계약 기한 |
| `user_id` | UUID | NULL, UNIQUE | 유효성 확인 뒤 발급하는 예정 계정 ID. 계정 생성보다 먼저 저장하므로 FK를 두지 않는다. |
| `link_request_id` | VARCHAR(128) | NOT NULL, UNIQUE | 사용자 생성 전 거부에도 필요한 업무 멱등 키 |
| `link_event_id` | VARCHAR(128) | NULL, UNIQUE | Auth 결과 causation 대조용 발행 Event ID |
| `account_command_id` | VARCHAR(128) | NULL, UNIQUE | 계정 생성 Command 멱등 키 |
| `profile_command_id` | VARCHAR(128) | NULL, UNIQUE | 기본 프로필 생성 Command 멱등 키 |
| `account_created` | BOOLEAN | NOT NULL DEFAULT false | 계정 준비 상태 |
| `profile_created` | BOOLEAN | NOT NULL DEFAULT false | 프로필 준비 상태 |
| `link_status` | VARCHAR(32) | NOT NULL, CHECK | not_requested/requested/linked/rejected/timed_out |
| `auth_result_version` | BIGINT | NULL, CHECK `> registration_version` | 마지막으로 반영한 Auth 결과 version |
| `failure_code` | VARCHAR(100) | NULL | 일반화된 업무 결과 |
| `version` | BIGINT | NOT NULL | Process Manager version |
| `created_at`, `updated_at` | TIMESTAMPTZ | NOT NULL | 서버 시각 |

`received_at`은 사용자 서비스의 로컬 처리 여유를 판단하는 보조 시각이다. 현재 시각에 clock skew와 broker transport budget을 더한 값이 `link_accept_until`을 넘으면 새 계정 생성을 시작하지 않는다. 최종 수락은 Auth Inbox 수신 시각으로 Auth가 판정하며, 이미 시작한 로컬 grace가 그 기한을 연장하지 않는다.

`profile_request_id`와 `user_id`의 FK를 의도적으로 두지 않는다. 전자는 존재하지 않는 초안에 대한 멱등 거부를 기록해야 하고, 후자는 Process Manager가 계정보다 먼저 예정 ID를 저장해야 한다. 유효한 초안의 실제 소비는 `user_registration_drafts.consumed_by_provisioning_id`, 생성된 계정의 귀속은 `user_accounts.provisioning_id` FK와 reconciliation으로 확인한다.

`status`와 `link_status`는 위 허용값만 받는 CHECK를 둔다. `auth_result_version`은 NULL이 아니면 최초 `registration_version`보다 커야 하고 더 낮은 값으로 갱신할 수 없다. Event에 제시된 `agreement_receipt_id` 자체는 invalid 거부를 멱등 기록하기 위해 Provisioning에 저장한다. Agreement Port가 purpose=`user_registration`, 동일 `registration_process_id`와 `profile_request_id`, 필수 약관 version과 유효 시각을 확인한 경우에만 status를 `valid`로 두고 계정 생성을 진행한다. invalid이면 binding hash를 저장하지 않고 Provisioning과 존재하는 draft를 `rejected`로 닫는다.

```sql
CHECK (link_status IN ('not_requested', 'requested', 'linked', 'rejected', 'timed_out')),
CHECK (auth_result_version IS NULL OR auth_result_version > registration_version),
CHECK (agreement_validation_status IN ('pending', 'valid', 'invalid')),
CHECK (
  agreement_validation_status <> 'valid' OR agreement_binding_hash IS NOT NULL
)
```

## user_accounts

| 필드 | 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `user_id` | UUID | PK | 불변 사용자 ID |
| `provisioning_id` | VARCHAR(128) | NOT NULL, UNIQUE, FK user_provisionings | 가입 작업 참조 |
| `lifecycle_status` | VARCHAR(32) | NOT NULL | provisioning/active/restricted/deactivated/provisioning_failed |
| `auth_status` | VARCHAR(32) | NOT NULL | active/restricted/deactivated |
| `restriction_version` | BIGINT | NOT NULL, CHECK `> 0` | Auth projection version |
| `account_version` | BIGINT | NOT NULL, CHECK `> 0` | 전체 계정 version |
| `status_reason_code` | VARCHAR(100) | NULL | 일반화된 사유 |
| `activated_at` | TIMESTAMPTZ | NULL | Auth link 성공 시각 |
| `deactivated_at` | TIMESTAMPTZ | NULL | 비활성 시각 |
| `created_at`, `updated_at` | TIMESTAMPTZ | NOT NULL | 서버 시각 |

DB trigger로 업무 전이를 숨기지 않는다. Handler가 도메인 전이를 검증한 뒤 `WHERE account_version = $expected` 조건으로 갱신한다.

다음 CHECK로 lifecycle/Auth 상태 대응을 방어한다.

```sql
CHECK (
  (lifecycle_status = 'restricted' AND auth_status = 'restricted') OR
  (lifecycle_status = 'deactivated' AND auth_status = 'deactivated') OR
  (lifecycle_status IN ('provisioning', 'active', 'provisioning_failed') AND auth_status = 'active')
)
```

상태 Handler는 두 열을 원자적으로 갱신한다. `auth_status`가 바뀌면 더 높은 `restriction_version`, 상태 이력, Auth 전달 Outbox도 같은 트랜잭션에 포함한다. `provisioning -> active`와 `provisioning -> provisioning_failed`는 Auth 상태를 바꾸지 않는다.

## user_profiles

| 필드 | 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `user_id` | UUID | PK, FK user_accounts ON DELETE RESTRICT | 같은 사용자 서비스 DB의 UserAccount를 참조한다. |
| `private_name_ciphertext` | BYTEA | NOT NULL | 사용자 private name |
| `private_name_key_version` | VARCHAR(64) | NOT NULL | 암호화 key version |
| `nickname` | VARCHAR(255) | NULL | unique 아님 |
| `nickname_policy_version` | VARCHAR(64) | NULL | nickname을 승인한 정책 version |
| `introduction` | VARCHAR(1000) | NULL | 정책 확정 전 DB 방어 상한 |
| `introduction_policy_version` | VARCHAR(64) | NULL | introduction을 승인한 정책 version |
| `profile_media_asset_id` | VARCHAR(128) | NULL | 검증된 opaque media ID |
| `profile_version` | BIGINT | NOT NULL, CHECK `> 0` | 요청 본문 expected version 기반 낙관적 제어 |
| `created_at`, `updated_at` | TIMESTAMPTZ | NOT NULL | 서버 시각 |

nickname/introduction 값과 해당 policy version은 함께 NULL이거나 함께 NOT NULL이어야 한다. Profile을 재수화할 때 승인 정책 version을 잃지 않도록 CHECK로 두 쌍의 일관성을 방어한다.

## user_account_status_history

| 필드 | 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `status_change_id` | UUID | PK | 변경 식별자 |
| `user_id` | UUID | NOT NULL, FK user_accounts ON DELETE RESTRICT | 대상 사용자 |
| `from_status`, `to_status` | VARCHAR(32) | NOT NULL | 상태 전이 |
| `restriction_version` | BIGINT | NOT NULL | 변경 뒤 version |
| `reason_code` | VARCHAR(100) | NOT NULL | 일반화 코드 |
| `actor_type` | VARCHAR(32) | NOT NULL | user/operator/system |
| `actor_ref` | VARCHAR(128) | NOT NULL | 내부 참조; event에는 미포함 |
| `approval_ref` | VARCHAR(128) | NULL | 고위험 운영 변경 승인 |
| `effective_at`, `created_at` | TIMESTAMPTZ | NOT NULL | 효력·기록 시각 |

UNIQUE `(user_id, restriction_version)`으로 순서를 보장한다. UPDATE/DELETE를 금지하고 정정은 새 상태 변경으로 남긴다.

## Repository

| Repository | 주요 메서드 | 잠금/조건 |
| --- | --- | --- |
| RegistrationDraftRepository | Create, FindForUpdate, Consume, Expire | profile_request_id, status=pending |
| UserProvisioningRepository | FindBySourceEvent, FindByAuthRegistration, Save | version 조건, 필요 시 FOR UPDATE |
| UserAccountRepository | Create, FindByID, FindByIDForUpdate, TransitionStatus | account_version/restriction_version |
| UserProfileRepository | Create, FindByUserID, UpdateOwnProfile | profile_version 조건 |
| AccountStatusHistoryRepository | Append, ListByUser | append-only |

범용 Repository와 transaction manager를 만들지 않는다. 구현은 pgx query와 `pgx.Tx`를 직접 사용하고 도메인별 Repository만 둔다.

## 확인 필요

- 실제 nickname/introduction DB 상한과 정책 값
- private name key rotation과 재암호화 batch
- status history 법적 보존 기간
- provisioning_failed profile의 crypto-shred 시점
