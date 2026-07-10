---
id: SD.A.30020
title: Context 인증 영속성 설계
type: service-design-persistence
status: draft
tags: [service-design, auth, persistence, postgresql, repository, outbox]
source: local
created: 2026-07-09
updated: 2026-07-10
service_design: SD.A.300
bounded_context: BC.A.300
domain_model: SD.A.30010
service: SD.A.30030
api: SD.A.30040
requirements: [REQ.A.05]
use_cases: [UC.A.300]
---

# Context 인증 영속성 설계

## 역할

Context 인증의 Aggregate와 처리 상태를 PostgreSQL에 저장하고, 인증 식별자 단일 연결, 로그인 잠금, 세션 회전, 권한 claim 스냅샷, 비밀번호 재설정, outbox 전송을 데이터베이스 제약과 트랜잭션으로 보장한다.

이 문서에서 인증 Use Case의 기준 식별자는 `UC.A.300`이다. 요구사항 원천 식별자 `REQ.A.05`를 Use Case 식별자로 재사용하지 않는다.

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) | UC 참조: [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) | BC 참조: [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) | 도메인 참조: [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) | 서비스 참조: [SD.A.30030](../A_300_30-service/README.md) | API 참조: [SD.A.30040](../A_300_40-api/README.md)

## 저장 경계

### 소유 데이터

- Context 인증은 `Identity`, `PasswordCredential`, `VerificationChallenge`, `IdentityLink`, `Registration`, `PasswordReset`, `AuthenticationIntent`, `Session`, `SessionCredential`, `AccessGrant`, 외부 projection인 `UserAuthState`의 현재 상태와 변경 이력을 저장한다. 고위험 작업의 단기 `ReauthenticationProof`는 1회 소비를 보장하는 보안 레코드로 저장한다.
- `user_id`는 Context 사용자가 발급한 외부 식별자다. 인증 DB는 값만 참조하며 다른 컨텍스트 DB에 외래 키를 만들지 않는다.
- 역할과 권한은 access token에 넣기 위한 `AccessGrant` 스냅샷으로 저장한다. 사용자 표시명, 주소, 연락처 프로필, 마케팅 속성은 저장하지 않는다.
- 감사 이벤트의 검색 원장은 Context 감사가 소유한다. 인증 DB에는 전송 보장을 위한 outbox 레코드만 둔다.

### 저장하지 않는 데이터

| 데이터 | 처리 원칙 |
| --- | --- |
| 비밀번호 원문 | 메모리에서 검증한 뒤 즉시 버린다. 로그, trace, idempotency 결과, outbox에 기록하지 않는다. |
| 이메일 인증 token, SMS 인증번호 평문 | Challenge에는 keyed hash만 저장한다. 발송에 필요한 원문은 전용 short-TTL delivery ciphertext에만 저장한다. |
| 웹 session cookie, mobile refresh token 평문 | SessionCredential에는 keyed hash만 저장한다. refresh 멱등 재생은 전용 short-TTL replay ciphertext만 예외로 허용한다. |
| ReauthenticationProof 원문 | proof 식별자와 secret을 분리하고 keyed hash만 저장한다. 목적, Session, 만료·소비 상태만 단기 보존한다. |
| access JWT, internal context JWT 평문 | 짧은 수명의 발급 결과물이다. 기본적으로 `Session`과 `AccessGrant`만 저장하며, access JWT는 refresh replay 전용 short-TTL ciphertext에만 예외적으로 포함한다. |
| 이메일·휴대폰 원문 평문 | 정규화 값은 envelope encryption한 ciphertext로 저장하고, 정확 일치 조회는 별도 HMAC으로 처리한다. |
| 이름, 추천인 코드, 약관 동의, 프로필 | BFF와 담당 Context가 직접 처리한다. 인증 DB에는 담당 Context가 발급한 opaque 참조만 저장한다. |
| 감사 검색 원장 | Context 감사가 보관한다. 인증 DB에는 전달 전 outbox만 유지한다. |

## 기술 기준

- 기준 저장소는 PostgreSQL이다. Redis를 사용하더라도 세션과 권한 상태의 최종 기준은 PostgreSQL이다.
- 시각은 모두 UTC `TIMESTAMPTZ`로 저장한다. 만료 판정은 애플리케이션 서버 시각이 아니라 DB 트랜잭션 안의 `transaction_timestamp()`를 기준으로 한다.
- 내부 식별자는 `UUID`다. 애플리케이션에서 생성하며 값에 업무 의미를 넣지 않는다. 쓰기 지역성이 필요한 구현에서는 UUIDv7을 우선 검토한다.
- 상태 값은 PostgreSQL enum 대신 `VARCHAR`와 `CHECK`를 사용한다. 상태 추가 시 enum 타입 교체 없이 expand/contract migration을 적용하기 위해서다.
- Aggregate Root에는 `row_version BIGINT`를 두고 낙관적 잠금을 적용한다. 잠금 횟수 증가, token 회전, 번호 교체처럼 경합이 예상되는 명령은 `SELECT ... FOR UPDATE`를 함께 사용한다.
- 인증 스키마의 테이블은 `auth_` 접두사를 사용한다. 모든 외래 키는 별도 명시가 없으면 `ON DELETE RESTRICT`다.

## 저장 모델

| 저장 모델 | 역할 | 연결 도메인 | 기준 보존 방식 |
| --- | --- | --- | --- |
| `auth_policy_versions` | 배포 없이 바꾸는 로그인 잠금, token TTL, rotation 정책의 불변 버전 | `LoginLockPolicy`, `TokenTtlPolicy`, `RefreshRotationPolicy` | 활성 버전 1개, 이전 버전 보존 |
| `auth_verification_policy_rules` | 정책 버전별 purpose/channel 인증 제한 | `VerificationPolicy` | 정책 버전 하위 불변 규칙 |
| `auth_session_revocation_policy_rules` | 보안 사건별 Session 폐기 범위 | `SessionRevocationPolicy` | 정책 버전 하위 불변 규칙 |
| `auth_identities` | 인증 식별자 값, 소유 확인 상태, 로그인 잠금과 닫힘 상태 | `Identity` | Aggregate Root |
| `auth_password_credentials` | 이메일 Identity의 비밀번호 verifier | `PasswordCredential` | Identity별 active 1개 |
| `auth_verification_challenges` | 이메일 link와 SMS 인증번호의 발급·검증·만료 상태 | `VerificationChallenge` | challenge별 독립 Aggregate |
| `auth_verification_delivery_payloads` | provider 발송에 필요한 목적지와 code/link의 단기 암호문 | 발송 인프라 레코드 | challenge send별 1개, terminal 처리 후 crypto-shred |
| `auth_identity_links` | Identity와 `user_id` 연결 및 교체 이력 | `IdentityLink` | Aggregate Root |
| `auth_registrations` | 이메일·휴대폰 검증부터 사용자 생성, 연결, 자동 로그인 준비까지의 장기 작업 | `Registration` | Process Aggregate |
| `auth_password_resets` | 이메일 또는 휴대폰 증명 후 비밀번호 교체까지의 상태 | `PasswordReset` | Process Aggregate |
| `auth_action_intent_payloads` | 로그인 뒤 재개할 action의 최소 payload 단기 암호문 | AuthenticationIntent 하위 보안 레코드 | Intent별 최대 1개, 전달/만료 뒤 crypto-shred |
| `auth_authentication_intents` | 로그인 전 검증된 내부 복귀 위치와 의도 | `AuthenticationIntent` | 1회 소비 Aggregate |
| `auth_sessions` | 사용자 로그인 상태와 만료·폐기·재사용 탐지 상태 | `Session` | Aggregate Root |
| `auth_session_credentials` | cookie/refresh token 검증, 회전, 재사용 탐지 | `SessionCredential` | Session 하위 Entity |
| `auth_access_grants` | `user_id`별 role/permission grant와 claim version | `AccessGrant` | 사용자별 active 1개, Session은 발급 snapshot 참조 |
| `auth_user_auth_states` | Context 사용자 제한·비활성 version과 인증 허용 상태 | `UserAuthState` | user_id별 1행, source version 단조 증가 |
| `auth_reauthentication_proofs` | 고위험 작업용 proof의 목적 바인딩과 1회 소비 상태 | `ReauthenticationProof` 보안 레코드 | 짧은 TTL, proof별 1회 소비 |
| `auth_idempotency_records` | 중복 Command와 외부 이벤트의 처리 결과 참조 | `IdempotencyRecord` | 만료되는 인프라 레코드 |
| `auth_idempotency_replay_payloads` | refresh 최초 응답 token 묶음의 단기 암호문 | 멱등 재생 인프라 레코드 | IdempotencyRecord별 최대 1개 |
| `auth_outbox_events` | Domain Event와 감사 이벤트의 at-least-once 전달 | `OutboxEvent` | append-only 인프라 레코드 |

## 스키마

### `auth_policy_versions`

정책 행은 활성화된 뒤 수정하지 않는다. 정책 변경은 새 버전을 insert하고 이전 버전을 `superseded`로 닫는 방식으로 적용한다. 각 인증 레코드는 자신에게 실제로 적용된 `policy_version`을 참조하므로 운영 중 정책이 바뀌어도 과거 판정을 설명할 수 있다.

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `policy_version` | BIGINT | PK, GENERATED ALWAYS AS IDENTITY | 불변 정책 버전 |
| `status` | VARCHAR(16) | NOT NULL, CHECK | `active`, `superseded` |
| `login_failure_threshold` | SMALLINT | NOT NULL, CHECK `> 0`, 기본 5 | 잠금 전 허용 실패 횟수 |
| `login_failure_window_seconds` | INTEGER | NOT NULL, CHECK `> 0` | 실패 횟수를 합산하는 구간 |
| `lock_duration_seconds` | INTEGER | NOT NULL, CHECK `> 0` | 잠금 유지 시간 |
| `reset_failure_on_success` | BOOLEAN | NOT NULL, 기본 true | 로그인 성공 시 실패 구간 초기화 여부 |
| `web_idle_ttl_seconds` | INTEGER | NOT NULL, CHECK `> 0` | 웹 Session 유휴 TTL |
| `web_absolute_ttl_seconds` | INTEGER | NOT NULL, CHECK `>= web_idle_ttl_seconds` | 웹 Session 최대 TTL |
| `access_ttl_seconds` | INTEGER | NOT NULL, CHECK `> 0`, 기본 900 | access JWT TTL |
| `refresh_ttl_seconds` | INTEGER | NOT NULL, CHECK `> access_ttl_seconds`, 기본 1,209,600 | 일반 refresh token TTL |
| `remember_me_ttl_seconds` | INTEGER | NOT NULL, CHECK `>= refresh_ttl_seconds`, 기본 2,592,000 | 로그인 상태 유지 TTL |
| `internal_context_ttl_seconds` | INTEGER | NOT NULL, CHECK `> 0` | 내부 context token TTL |
| `refresh_rotation_enabled` | BOOLEAN | NOT NULL, 기본 true | refresh token 회전 여부 |
| `refresh_reuse_action` | VARCHAR(32) | NOT NULL, CHECK | MVP는 `revoke_family_and_session` |
| `activation_source` | VARCHAR(16) | NOT NULL, CHECK | 초기 `bootstrap` 또는 운영자 변경 `operator` |
| `activated_by_user_id` | UUID | NULL | 운영자 변경일 때 정책을 활성화한 `user_id` |
| `change_reason` | VARCHAR(500) | NOT NULL | 변경 사유. 프로필 정보는 넣지 않는다. |
| `effective_at` | TIMESTAMPTZ | NOT NULL | 적용 시작 시각 |
| `superseded_at` | TIMESTAMPTZ | NULL | 다음 버전으로 교체된 시각 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 생성 시각 |

`lock_duration_seconds`의 초기값은 운영 확인 후 첫 정책 seed에 명시한다. 값이 없는 활성 정책은 허용하지 않는다. `activation_source = 'operator'`이면 `activated_by_user_id`가 필요하고, `bootstrap`이면 비워 둔다.

`auth_verification_policy_rules`는 `(policy_version, purpose, channel)`을 PK로 사용하고 `ttl_seconds`, `max_attempts`, `max_sends`, `resend_interval_seconds`를 모두 양수로 저장한다. `auth_session_revocation_policy_rules`는 `(policy_version, trigger, scope)`를 PK로 사용하고 scope를 `current_session`, `identity_sessions`, `user_sessions`, `refresh_family` 중 하나로 제한한다. 한 trigger에 여러 scope를 둘 수 있다. 정책 활성화 transaction은 두 하위 규칙 집합이 모두 유효한지 확인한 뒤 버전 전체를 active로 바꾼다.

`trigger = 'password_reset'`에는 `user_sessions`와 `refresh_family`가 모두 있어야 하며 이보다 좁은 scope만 있는 정책 버전은 활성화할 수 없다. 비밀번호 재설정 완료 transaction은 정책 설정과 관계없이 이 최소 범위를 강제한다.

| 하위 저장 모델 | 필드 | PostgreSQL 타입 | 제약 |
| --- | --- | --- | --- |
| `auth_verification_policy_rules` | `policy_version` | BIGINT | PK 일부, FK |
| `auth_verification_policy_rules` | `purpose` | VARCHAR(32) | PK 일부 |
| `auth_verification_policy_rules` | `channel` | VARCHAR(24) | PK 일부 |
| `auth_verification_policy_rules` | `ttl_seconds`, `max_attempts`, `max_sends`, `resend_interval_seconds` | INTEGER | 모두 NOT NULL, CHECK `> 0` |
| `auth_session_revocation_policy_rules` | `policy_version` | BIGINT | PK 일부, FK |
| `auth_session_revocation_policy_rules` | `trigger` | VARCHAR(32) | PK 일부 |
| `auth_session_revocation_policy_rules` | `scope` | VARCHAR(32) | PK 일부, 허용 scope CHECK |

### `auth_identities`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `identity_id` | UUID | PK | Identity 식별자 |
| `identity_type` | VARCHAR(32) | NOT NULL, CHECK | `email`, `phone`, `provider_subject`, `passkey` |
| `identity_namespace` | VARCHAR(64) | NOT NULL | `email`, `phone` 또는 provider/RP 구분값 |
| `value_ciphertext` | BYTEA | NOT NULL | 정규화한 인증값의 envelope-encrypted ciphertext |
| `value_key_id` | VARCHAR(128) | NOT NULL | KMS key/DEK 버전 참조 |
| `value_lookup_hash` | BYTEA | NOT NULL, CHECK 32 bytes | 정확 일치 조회용 HMAC-SHA-256 |
| `lookup_key_version` | SMALLINT | NOT NULL | lookup HMAC key 버전 |
| `masked_value` | VARCHAR(320) | NOT NULL | UI/CS에서 사용하는 비가역 마스킹 표시값 |
| `verification_status` | VARCHAR(24) | NOT NULL, CHECK | `pending`, `verified`, `expired`, `failed` |
| `credential_status` | VARCHAR(32) | NOT NULL, CHECK | `active`, `locked`, `password_reset_required`, `revoked`, `superseded` |
| `owner_user_id` | UUID | NULL, 외부 참조 | 처음 active 연결할 때 고정하는 영구 귀속 사용자 |
| `failure_count` | INTEGER | NOT NULL, 기본 0, CHECK `>= 0` | 연속 로그인 실패 횟수 |
| `failure_window_started_at` | TIMESTAMPTZ | NULL | 현재 실패 집계 구간 시작 시각 |
| `lock_until` | TIMESTAMPTZ | NULL | 로그인 재허용 시각 |
| `lock_policy_version` | BIGINT | FK, NULL | 현재 잠금 판정에 적용한 정책 |
| `password_reset_required_at` | TIMESTAMPTZ | NULL | 강제 비밀번호 재설정 전환 시각 |
| `password_reset_reason` | VARCHAR(64) | NULL | 외부에 직접 노출하지 않는 정책 사유 코드 |
| `verified_at` | TIMESTAMPTZ | NULL | 소유 확인 완료 시각 |
| `superseded_by_identity_id` | UUID | self FK, NULL | 휴대폰 번호 교체 후 새 Identity |
| `row_version` | BIGINT | NOT NULL, 기본 0 | 낙관적 잠금 버전 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 생성 시각 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | 마지막 변경 시각 |

주요 `CHECK`는 다음과 같다.

- `credential_status = 'locked'`이면 `lock_until`과 `lock_policy_version`이 모두 필요하다.
- `credential_status = 'password_reset_required'`이면 `password_reset_required_at`과 reason이 필요하다. 이메일 Identity에만 허용하고 비밀번호 재설정 완료 transaction에서 active로 되돌린다.
- `credential_status = 'superseded'`이면 `superseded_by_identity_id`가 필요하고 자기 자신을 참조할 수 없다.
- `verification_status = 'verified'`이면 `verified_at`이 필요하다.
- `failure_count > 0`이면 `failure_window_started_at`이 필요하다.
- `owner_user_id`는 한 번 설정되면 NULL이나 다른 `user_id`로 바꿀 수 없다. active IdentityLink의 `user_id`는 이 값과 같아야 한다.
- 같은 `identity_type + identity_namespace + value_lookup_hash`는 닫힌 Identity를 포함해 하나만 허용한다. 전화번호 재할당처럼 동일 값 재사용이 필요하면 기존 Identity를 자동으로 새 계정에 연결하지 않고 별도 운영 정책을 거친다.
- `owner_user_id IS NULL`이고 IdentityLink가 없는 행은 가입 예약이다. active Registration이 있으면 다른 가입이 가져갈 수 없고, failed/expired Registration만 cooldown과 rate limit 뒤 같은 Identity 행으로 재예약할 수 있다.
- 미귀속 예약을 재사용할 때 이전 Registration과 Challenge를 다시 열지 않는다. 새 Registration을 만들고 새 Challenge를 발급하며, 이메일 PasswordCredential은 새 입력으로 교체한다. owner가 있거나 과거 Link가 있는 Identity는 이 규칙으로 재사용하지 않는다.

### `auth_password_credentials`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `password_credential_id` | UUID | PK | PasswordCredential 식별자 |
| `identity_id` | UUID | FK, NOT NULL | email Identity |
| `status` | VARCHAR(16) | NOT NULL, CHECK | `active`, `replaced`, `revoked` |
| `password_hash` | TEXT | NULL | active 상태에서만 존재하는 Argon2id PHC 문자열 |
| `hash_algorithm` | VARCHAR(24) | NOT NULL | `argon2id` |
| `hash_parameters` | JSONB | NOT NULL | 메모리, 반복 횟수, 병렬도, parameter policy 버전 |
| `credential_version` | INTEGER | NOT NULL, CHECK `> 0` | Identity 안에서 단조 증가하는 비밀번호 버전 |
| `changed_at` | TIMESTAMPTZ | NOT NULL | 설정/변경 시각 |
| `replaced_at` | TIMESTAMPTZ | NULL | 새 비밀번호로 교체된 시각 |
| `row_version` | BIGINT | NOT NULL, 기본 0 | 동시 교체 방지 |

- Identity별 active PasswordCredential은 하나다.
- `status = 'active'`이면 `password_hash`가 필요하다.
- `replaced` credential은 `replaced_at`이 필요하다.
- 교체된 credential은 같은 트랜잭션에서 `password_hash = NULL`로 지워 복구 가능한 이전 비밀번호 verifier를 남기지 않는다. 이력에는 ID, 알고리즘/parameter, credential version, 교체 시각만 남긴다.
- DB 제약만으로 Identity type을 확인할 수 없으므로 Repository가 email Identity인지 확인한 뒤 저장한다.

### `auth_verification_challenges`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `challenge_id` | UUID | PK | VerificationChallenge 식별자 |
| `purpose` | VARCHAR(32) | NOT NULL, CHECK | `signup_email`, `signup_phone`, `phone_signin`, `password_reset`, `phone_change`, `identity_link` |
| `channel` | VARCHAR(24) | NOT NULL, CHECK | `email_link`, `email_code`, `sms_code` |
| `target_lookup_hash` | BYTEA | NOT NULL, CHECK 32 bytes | 대상 IdentityLookupKey의 keyed hash |
| `target_lookup_key_version` | SMALLINT | NOT NULL | 대상 lookup HMAC key 버전 |
| `identity_id` | UUID | FK, NULL | 기존 Identity가 확인된 경우에만 내부 참조 |
| `context_type` | VARCHAR(32) | NOT NULL, CHECK | `registration`, `password_reset`, `identity_link`, `phone_signin`, `phone_change` |
| `context_id` | UUID | NOT NULL | 상위 작업의 opaque UUID |
| `status` | VARCHAR(16) | NOT NULL, CHECK | `issued`, `verified`, `failed`, `expired`, `revoked` |
| `verifier_hash` | BYTEA | NOT NULL, CHECK 32 bytes | `challenge_id + token/OTP` keyed hash |
| `verifier_key_version` | SMALLINT | NOT NULL | verifier HMAC key 버전 |
| `attempt_count` | SMALLINT | NOT NULL, 기본 0 | 검증 실패 횟수 |
| `max_attempts_snapshot` | SMALLINT | NOT NULL, CHECK `> 0` | 발급 당시 최대 검증 횟수 |
| `send_count` | SMALLINT | NOT NULL, 기본 0 | 발송 횟수 |
| `max_sends_snapshot` | SMALLINT | NOT NULL, CHECK `> 0` | 발급 당시 최대 발송 횟수 |
| `next_send_at` | TIMESTAMPTZ | NOT NULL | 다음 재발송 가능 시각 |
| `policy_version` | BIGINT | FK, NOT NULL | 발급 당시 VerificationPolicy 버전 |
| `expires_at` | TIMESTAMPTZ | NOT NULL | challenge 만료 시각 |
| `verified_at` | TIMESTAMPTZ | NULL | 검증 성공 시각 |
| `closed_at` | TIMESTAMPTZ | NULL | 실패/만료/폐기 시각 |
| `row_version` | BIGINT | NOT NULL, 기본 0 | 동시 검증 방지 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 생성 시각 |

- `issued` challenge만 검증할 수 있으며, 성공 시 `verified_at`이 필요하다.
- `attempt_count >= max_attempts_snapshot`이면 `failed`로 닫는다.
- 같은 context와 purpose에는 issued challenge를 하나만 둔다. 재발급은 기존 challenge를 `revoked`로 바꾸고 새 행을 만든다.
- purpose와 channel 조합은 해당 `policy_version`의 VerificationPolicy rule에 존재해야 한다.
- 이메일 link와 SMS 검증 요청은 `challenge_id`를 함께 받아 해당 행을 잠근 뒤 verifier를 비교한다. 짧은 SMS 번호의 hash만으로 challenge를 전역 조회하지 않는다.
- 발송 outbox에는 `challenge_id`와 `delivery_payload_ref`만 넣는다. relay는 아래 전용 payload를 전송 직전에만 복호화한다. Domain/Audit event와 일반 outbox payload에 목적지, code/link, ciphertext를 복제하지 않는다.

### `auth_verification_delivery_payloads`

HMAC만으로는 이메일 link나 SMS code를 복원할 수 없으므로, provider relay가 사용할 최소 발송 payload를 별도 short-TTL ciphertext로 보관한다.

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `delivery_payload_id` | UUID | PK | outbox의 `delivery_payload_ref` |
| `challenge_id` | UUID | FK, NOT NULL | 발송 대상 VerificationChallenge |
| `send_sequence` | SMALLINT | NOT NULL, CHECK `> 0` | Challenge 안의 발송 순번 |
| `payload_ciphertext` | BYTEA | NULL | destination, template/locale, code/link의 envelope ciphertext |
| `payload_key_id` | VARCHAR(128) | NOT NULL | 전용 short-lived encryption key 참조 |
| `aad_hash` | BYTEA | NOT NULL, CHECK 32 bytes | challenge, purpose, channel, send 순번 binding |
| `delivery_status` | VARCHAR(16) | NOT NULL, CHECK | `pending`, `delivered`, `failed`, `expired`, `destroyed` |
| `provider_request_id` | VARCHAR(128) | NULL | provider 응답의 비민감 참조 |
| `expires_at` | TIMESTAMPTZ | NOT NULL | Challenge 만료를 넘지 않는 payload TTL |
| `delivered_at` | TIMESTAMPTZ | NULL | provider 수락 시각 |
| `destroyed_at` | TIMESTAMPTZ | NULL | ciphertext 삭제/crypto-shred 시각 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 생성 시각 |

- `(challenge_id, send_sequence)`는 유일하며 `delivery_payload_id`를 provider idempotency key로 사용한다.
- Challenge, delivery payload, `VerificationDeliveryRequested` outbox를 한 transaction에 저장한다.
- relay는 transient 실패일 때 만료 전 같은 delivery ID로 재시도한다. provider 수락, 영구 실패, 만료 같은 terminal 상태가 되면 `payload_ciphertext`를 NULL 처리하거나 DEK를 폐기한다.
- payload ciphertext와 key ID는 Domain/Audit event, log, trace, metric에 넣지 않는다.

### `auth_identity_links`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `identity_link_id` | UUID | PK | IdentityLink 식별자 |
| `identity_id` | UUID | FK, NOT NULL | 연결할 Identity |
| `identity_type` | VARCHAR(32) | NOT NULL | 제약용 Identity type snapshot |
| `user_id` | UUID | NOT NULL, 외부 참조 | Context 사용자가 발급한 식별자 |
| `link_status` | VARCHAR(24) | NOT NULL, CHECK | `requested`, `active`, `replaced`, `revoked`, `manual_review` |
| `link_reason` | VARCHAR(24) | NOT NULL, CHECK | `signup`, `signin_link`, `phone_change`, `manual_operation` |
| `proof_challenge_id` | UUID | FK, NULL | 연결 소유 증명에 사용한 VerificationChallenge |
| `reauthentication_proof_id` | UUID | FK, NULL | requested intent 시작 시 소비한 목적 한정 proof |
| `previous_identity_link_id` | UUID | self FK, NULL | 교체 전 연결 |
| `requested_at` | TIMESTAMPTZ | NOT NULL | 연결 요청 시각 |
| `intent_expires_at` | TIMESTAMPTZ | NULL | requested link/phone replacement 완료 기한 |
| `activated_at` | TIMESTAMPTZ | NULL | active 전환 시각 |
| `closed_at` | TIMESTAMPTZ | NULL | replaced/revoked 전환 시각 |
| `closed_reason` | VARCHAR(64) | NULL | 운영/보안 닫힘 사유 코드 |
| `row_version` | BIGINT | NOT NULL, 기본 0 | 낙관적 잠금 버전 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 생성 시각 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | 마지막 변경 시각 |

`identity_type`은 검색 편의를 위한 임의 복제가 아니라 DB 제약을 위한 불변 snapshot이다. `(identity_id, identity_type)` 복합 외래 키로 `auth_identities`의 값과 일치시킨다.

- Identity 하나에는 active link가 하나만 존재한다.
- `linkIntentId`와 `replacementId`는 `link_status = 'requested'`인 `identity_link_id`다. requested에는 `reauthentication_proof_id`와 `intent_expires_at`이 필요하며, phone_change에는 기존 active phone Link를 가리키는 `previous_identity_link_id`도 필요하다.
- Challenge의 `context_type=identity_link|phone_change`, `context_id`는 이 requested IdentityLink ID를 가리킨다. proof 소비, requested Link 생성, 상위 Session/user 바인딩은 같은 transaction으로 처리한다.
- 만료 worker는 requested Link를 `revoked`와 `closed_reason=intent_expired`로 닫고 연결된 issued Challenge와 미소비 proof도 폐기한다. 새 Identity에 owner/다른 Link가 없으면 개인정보 cooldown 뒤 ciphertext와 lookup 예약을 정리한다.
- active link의 `user_id`는 Identity의 영구 `owner_user_id`와 같아야 한다. 첫 연결 transaction에서 owner와 link를 함께 설정한다.
- 같은 Identity의 모든 과거 link는 같은 `user_id`만 가질 수 있다.
- MVP의 `email`과 `phone`은 `user_id`별 active link를 각각 하나만 허용한다. 휴대폰 변경은 이전 link를 닫고 새 link를 같은 트랜잭션에서 연다.
- `replaced`, `revoked`이면 `closed_at`이 필요하다. 새 `phone_change` link는 닫힌 이전 link를 `previous_identity_link_id`로 가리키며, 이전 link에는 후속 link를 둘 이상 만들 수 없다.
- user 계정 병합을 표현하는 컬럼이나 명령은 두지 않는다.

### `auth_registrations`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `registration_id` | UUID | PK | Registration 식별자 |
| `status` | VARCHAR(32) | NOT NULL, CHECK | `pending_verification`, `requesting_user`, `linking`, `issuing_session`, `completed`, `failed`, `expired` |
| `email_identity_id` | UUID | FK, NOT NULL | 가입 이메일 Identity |
| `phone_identity_id` | UUID | FK, NOT NULL | 가입 휴대폰 Identity |
| `email_challenge_id` | UUID | FK, UNIQUE, NULL | 명시적 발급 요청으로 만든 현재 이메일 Challenge |
| `phone_challenge_id` | UUID | FK, UNIQUE, NULL | 명시적 발급 요청으로 만든 현재 휴대폰 Challenge |
| `user_id` | UUID | NULL, 외부 참조 | Context 사용자 생성 결과 |
| `user_creation_request_id` | UUID | UNIQUE, NOT NULL | Context 사용자에 보내는 멱등 요청 ID |
| `agreement_receipt_id` | VARCHAR(128) | NULL | 동의 담당 Context가 발급한 opaque 참조 |
| `profile_request_id` | VARCHAR(128) | NULL | 프로필 Context 요청의 opaque 참조 |
| `session_id` | UUID | FK, UNIQUE, NULL | 자동 로그인으로 정확히 한 번 발급한 Session |
| `authentication_intent_id` | UUID | FK, NULL | 가입 후 복귀할 의도 |
| `client_channel` | VARCHAR(16) | NOT NULL, CHECK | `web`, `mobile` |
| `remember_me` | BOOLEAN | NOT NULL, 기본 false | 자동 로그인 session 정책 입력 |
| `session_policy_version` | BIGINT | FK, NOT NULL | 자동 로그인 재시도에 고정할 정책 버전 |
| `failure_code` | VARCHAR(64) | NULL | 민감하지 않은 처리 실패 코드 |
| `expires_at` | TIMESTAMPTZ | NOT NULL | 가입 처리 만료 시각 |
| `completed_at` | TIMESTAMPTZ | NULL | 연결과 자동 로그인 준비 완료 시각 |
| `row_version` | BIGINT | NOT NULL, 기본 0 | process 동시 실행 방지 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 시작 시각 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | 마지막 변경 시각 |

- email Identity와 phone Identity는 서로 달라야 하며 각각 올바른 type이어야 한다.
- 현재 Registration의 두 Challenge가 모두 존재하고 Challenge와 Identity가 verified이기 전에는 `requesting_user`로 전환할 수 없다.
- `user_id`가 없으면 `linking`, `issuing_session`, `completed`가 될 수 없다.
- `issuing_session`은 두 Identity가 같은 `user_id`를 owner로 갖고 두 link가 모두 active일 때만 허용한다.
- `completed`이면 두 Identity 모두 같은 `user_id`에 active 연결되고 `session_id`가 있어야 한다. 이 조건은 여러 테이블을 조회하므로 서비스 트랜잭션과 유니크 인덱스로 보장한다.
- 이름, 추천인 코드, 약관 동의 payload는 이 테이블에 저장하지 않는다. Context 사용자 명령의 별도 계약을 따른다.

### `auth_password_resets`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `password_reset_id` | UUID | PK | PasswordReset 식별자 |
| `status` | VARCHAR(24) | NOT NULL, CHECK | `requested`, `challenge_verified`, `completed`, `expired`, `revoked` |
| `identity_id` | UUID | FK, NULL | 존재하는 경우 비밀번호를 교체할 email Identity |
| `authentication_intent_id` | UUID | FK, NULL | 사전 인증 소유 증명과 완료 후 복귀에 사용하는 Intent |
| `challenge_id` | UUID | FK, UNIQUE, NULL | 사용자가 선택한 VerificationChallenge |
| `reset_grant_hash` | BYTEA | NULL, CHECK 32 bytes | Challenge 완료 후 발급하는 1회용 reset grant HMAC |
| `reset_grant_key_version` | SMALLINT | NULL | reset grant HMAC key 버전 |
| `policy_version` | BIGINT | FK, NOT NULL | reset 시작 시 정책 버전 |
| `expires_at` | TIMESTAMPTZ | NOT NULL | reset 처리 만료 시각 |
| `challenge_verified_at` | TIMESTAMPTZ | NULL | 선택한 Challenge 성공 시각 |
| `completed_at` | TIMESTAMPTZ | NULL | 새 비밀번호 적용 시각 |
| `row_version` | BIGINT | NOT NULL, 기본 0 | 중복 완료 방지 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 시작 시각 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | 마지막 변경 시각 |

- 계정 존재 여부를 숨기기 위해 `identity_id`와 `challenge_id`가 없는 decoy reset도 같은 외부 응답으로 만들 수 있다.
- `challenge_verified`이면 `challenge_id`, `reset_grant_hash`, `reset_grant_key_version`, `challenge_verified_at`이 모두 필요하다.
- reset grant는 `password_reset_id + identity_id + purpose + secret`에 묶어 HMAC하고 원문을 저장하지 않는다.
- 동일 `identity_id`에는 진행 중 reset을 하나만 둔다. NULL identity decoy는 이 제약 대상에서 제외한다.
- 완료 시 active PasswordCredential을 가진 email Identity인지 다시 확인하고 reset grant를 한 번 소비한다. 기존 PasswordCredential 교체, 해당 `user_id`의 전체 Session과 refresh family 폐기, outbox 저장은 하나의 트랜잭션이다.
- 완료 transaction은 소비한 `reset_grant_hash`를 NULL로 지우고 완료 시각만 남긴다.

### `auth_action_intent_payloads`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `action_payload_id` | UUID | PK | AuthenticationIntent의 opaque payload 참조 |
| `action_name` | VARCHAR(64) | NOT NULL | allowlist schema 이름 |
| `schema_version` | SMALLINT | NOT NULL | action payload 계약 버전 |
| `payload_ciphertext` | BYTEA | NULL | 최소 action payload의 envelope ciphertext |
| `payload_key_id` | VARCHAR(128) | NOT NULL | 전용 encryption key 참조 |
| `expires_at` | TIMESTAMPTZ | NOT NULL | 상위 Intent 만료를 넘지 않는 TTL |
| `delivered_at` | TIMESTAMPTZ | NULL | 로그인 성공 뒤 BFF에 전달 완료한 시각 |
| `destroyed_at` | TIMESTAMPTZ | NULL | ciphertext crypto-shred 시각 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 생성 시각 |

- `action_name`별 allowlist schema를 통과한 `dropId`, `optionId`, `quantity` 같은 최소값만 저장한다. URL, script, 결제 비밀정보, 개인정보 임의 JSON은 거부한다.
- AuthenticationIntent와 payload를 같은 transaction에서 만들고 Intent별 payload는 최대 하나다.
- 로그인/가입 완료 transaction이 Intent를 소비하면 payload를 같은 Session에 전달할 수 있게 한다. 최초 action-resume은 `delivered_at`, IdempotencyRecord, `auth.action_resume.delivered` 감사 outbox를 함께 저장하고 같은 key에는 짧은 delivery replay TTL 동안 같은 결과를 재구성한다. TTL 뒤 ciphertext를 crypto-shred한다. outbox와 로그에는 action name/result만 넣고 ciphertext/key ID를 넣지 않는다.

### `auth_authentication_intents`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `authentication_intent_id` | UUID | PK | AuthenticationIntent 식별자 |
| `owner_proof_hash` | BYTEA | UNIQUE, NOT NULL, CHECK 32 bytes | 사전 인증 cookie/mobile flow token의 keyed hash |
| `owner_proof_key_version` | SMALLINT | NOT NULL | owner proof HMAC key 버전 |
| `csrf_secret_hash` | BYTEA | NULL, CHECK 32 bytes | 웹 unsafe method용 CSRF secret의 keyed hash |
| `csrf_key_version` | SMALLINT | NULL | CSRF HMAC key 버전 |
| `status` | VARCHAR(16) | NOT NULL, CHECK | `active`, `consumed`, `expired` |
| `target_path` | VARCHAR(1024) | NOT NULL | 검증된 내부 경로 |
| `action_name` | VARCHAR(64) | NULL | 복귀 후 재개할 허용 action |
| `action_payload_ref` | UUID | FK, UNIQUE, NULL | `auth_action_intent_payloads` 참조 |
| `client_channel` | VARCHAR(16) | NOT NULL, CHECK | `web`, `mobile` |
| `consumed_by_session_id` | UUID | FK, NULL | 소비 결과 Session |
| `consumption_reason` | VARCHAR(32) | NULL, CHECK | `session_issued`, `password_reset_completed`, `cancelled` |
| `expires_at` | TIMESTAMPTZ | NOT NULL | 의도 만료 시각 |
| `consumed_at` | TIMESTAMPTZ | NULL | 1회 소비 시각 |
| `row_version` | BIGINT | NOT NULL, 기본 0 | 중복 소비 방지 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 생성 시각 |

- `target_path`는 `/`로 시작하는 내부 상대 경로만 저장한다. URL decode, host, scheme, allowlist 검증은 저장 전에 서비스가 완료한다.
- action payload 전체를 이 테이블의 JSON으로 저장하지 않는다. 허용된 `action_name`과 최소화·암호화한 payload의 opaque reference만 저장한다.
- owner proof와 CSRF secret 원문은 저장하지 않고 Intent ID, client channel, TTL을 HMAC 입력에 묶는다. web channel의 unsafe action에는 `csrf_secret_hash`와 key version이 필요하다.
- active이고 만료되지 않은 intent만 한 번 소비할 수 있다.
- consumed이면 `consumption_reason`이 필요하다. `session_issued`에는 `consumed_by_session_id`가 필요하고, password reset 완료/취소는 Session 없이 owner proof와 CSRF를 종료한다.

### `auth_sessions`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `session_id` | UUID | PK | Session 식별자 |
| `user_id` | UUID | NOT NULL, 외부 참조 | 로그인 사용자 |
| `identity_id` | UUID | FK, NOT NULL | 로그인에 사용한 Identity |
| `identity_link_id` | UUID | FK, NOT NULL | 로그인 당시 active IdentityLink |
| `authentication_method` | VARCHAR(32) | NOT NULL, CHECK | `registration_verified`, `email_password`, `phone_otp`, `provider`, `passkey` |
| `last_authenticated_at` | TIMESTAMPTZ | NOT NULL | 로그인 또는 최근 강한 재인증 시각 |
| `access_grant_id` | UUID | FK, NOT NULL | 발급 당시 active AccessGrant |
| `access_grant_version` | BIGINT | NOT NULL, CHECK `> 0` | claim에 반영한 grant version |
| `session_status` | VARCHAR(24) | NOT NULL, CHECK | `active`, `expired`, `revoked`, `reuse_detected` |
| `client_channel` | VARCHAR(16) | NOT NULL, CHECK | `web`, `mobile` |
| `remember_me` | BOOLEAN | NOT NULL | 로그인 상태 유지 여부 |
| `token_policy_version` | BIGINT | FK, NOT NULL | 발급 시 적용한 TTL/rotation 정책 |
| `issued_at` | TIMESTAMPTZ | NOT NULL | 발급 시각 |
| `idle_expires_at` | TIMESTAMPTZ | NULL | 웹 Session 유휴 만료 시각 |
| `absolute_expires_at` | TIMESTAMPTZ | NOT NULL | Session 최대 만료 시각 |
| `last_seen_at` | TIMESTAMPTZ | NULL | 쓰기 증폭을 제한해 갱신하는 마지막 사용 시각 |
| `revoked_at` | TIMESTAMPTZ | NULL | 폐기 시각 |
| `revocation_reason` | VARCHAR(64) | NULL | `logout`, `password_reset`, `refresh_reuse`, `operator` 등 |
| `reuse_detected_at` | TIMESTAMPTZ | NULL | credential 재사용 탐지 시각 |
| `row_version` | BIGINT | NOT NULL, 기본 0 | 회전/로그아웃 경합 방지 |

- active session만 credential을 사용할 수 있다.
- `absolute_expires_at > issued_at`이어야 한다. 웹 Session은 `idle_expires_at`이 필요하고 `absolute_expires_at`을 넘을 수 없다.
- Session의 `user_id`는 IdentityLink와 AccessGrant의 `user_id`와 같아야 하고 저장한 두 참조는 발급·재바인딩 시점에 active여야 한다.
- 강한 재인증이 다른 active IdentityLink를 사용하면 같은 `user_id` 안에서만 Session의 identity/link, `authentication_method`, `last_authenticated_at`을 함께 재바인딩할 수 있다. 사용자 간 재바인딩은 금지한다.
- `revoked`이면 `revoked_at`과 `revocation_reason`이 필요하다.
- `reuse_detected`이면 `reuse_detected_at`이 필요하고 credential 발급을 금지한다.

### `auth_session_credentials`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `session_credential_id` | UUID | PK | SessionCredential 식별자 |
| `session_id` | UUID | FK, NOT NULL | 소속 Session |
| `credential_type` | VARCHAR(32) | NOT NULL, CHECK | `web_session_cookie`, `mobile_refresh_token` |
| `credential_status` | VARCHAR(24) | NOT NULL, CHECK | `active`, `rotated`, `expired`, `revoked`, `reuse_detected` |
| `secret_hash` | BYTEA | UNIQUE, NOT NULL, CHECK 32 bytes | 고엔트로피 credential의 keyed hash |
| `secret_key_version` | SMALLINT | NOT NULL | credential HMAC key 버전 |
| `csrf_key_version` | SMALLINT | NULL | 웹 Session-bound CSRF token 도출 키 버전 |
| `refresh_family_id` | UUID | NULL | mobile refresh rotation 묶음 |
| `rotated_from_credential_id` | UUID | self FK, UNIQUE, NULL | 바로 이전 credential |
| `rotated_to_credential_id` | UUID | self FK, UNIQUE, NULL | 바로 다음 credential |
| `issued_at` | TIMESTAMPTZ | NOT NULL | 발급 시각 |
| `expires_at` | TIMESTAMPTZ | NOT NULL | 만료 시각 |
| `rotated_at` | TIMESTAMPTZ | NULL | 회전 시각 |
| `revoked_at` | TIMESTAMPTZ | NULL | 폐기 시각 |
| `reuse_detected_at` | TIMESTAMPTZ | NULL | 재사용 탐지 시각 |
| `row_version` | BIGINT | NOT NULL, 기본 0 | 동시 회전 방지 |

- `mobile_refresh_token`에는 `refresh_family_id`가 필요하고, `web_session_cookie`에는 비워 둔다.
- `web_session_cookie`에는 `csrf_key_version`이 필요하고 `mobile_refresh_token`에는 비워 둔다. CSRF token은 `HMAC(csrf_key_version, session_credential_id || secret_hash || "csrf")`로 도출해 원문을 저장하지 않는다.
- session과 credential의 `client_channel`/`credential_type` 조합은 Repository가 검증한다.
- session별 credential type과 refresh family별 active credential은 각각 하나다.
- credential의 `expires_at`은 소속 Session의 `absolute_expires_at`을 넘을 수 없다.
- `rotated`이면 `rotated_at`과 `rotated_to_credential_id`가 필요하다. 회전 연결은 순환할 수 없다.
- 회전은 새 credential UUID를 먼저 생성하고 한 transaction에서 기존 active 행을 `rotated`와 `rotated_to=new_id`로 갱신한 뒤 새 active 행을 `rotated_from=old_id`로 insert한다. self FK는 deferred이므로 commit 시점에 두 참조를 함께 검증하고, active partial unique index는 기존 행을 먼저 닫아 충돌하지 않는다.
- SessionCredential에는 token 원문을 저장하지 않는다. refresh의 동일 멱등 요청 재생만 전용 short-TTL encrypted replay payload에 격리한다.

### `auth_user_auth_states`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `user_id` | UUID | PK, 외부 참조 | Context 사용자 식별자 |
| `status` | VARCHAR(24) | NOT NULL, CHECK | `active`, `restricted`, `deactivated` |
| `restriction_version` | BIGINT | NOT NULL, CHECK `> 0` | 원천의 단조 증가 version |
| `reason_code` | VARCHAR(64) | NULL | 외부에 직접 노출하지 않는 일반 사유 |
| `effective_at` | TIMESTAMPTZ | NOT NULL | 효력 시각 |
| `source_event_id` | UUID | UNIQUE, NOT NULL | 멱등 소비 key |
| `row_version` | BIGINT | NOT NULL, 기본 0 | 동시 반영 제어 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | 마지막 반영 시각 |

- 제한/비활성 event는 더 높은 `restriction_version`만 반영한다. restricted/deactivated 전환과 active AccessGrant·전체 Session 폐기, cache invalidation outbox를 한 transaction에 저장한다.
- 로그인, refresh, 재인증은 UserAuthState가 없으면 Context 사용자/권한 원천의 초기 active 결과를 확정한 뒤 행을 만들고, 행이 있으면 반드시 active인지 확인한다. 장애나 cache miss를 active로 간주하지 않는다.
- active 복귀는 Context 사용자의 명시적 해제 event만 허용한다.

### `auth_access_grants`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `access_grant_id` | UUID | PK | AccessGrant 식별자 |
| `user_id` | UUID | NOT NULL, 외부 참조 | grant와 claim의 subject |
| `roles` | TEXT[] | NOT NULL | 정렬·중복 제거된 role 이름 |
| `permissions` | TEXT[] | NOT NULL | 정렬·중복 제거된 permission 이름 |
| `grant_version` | BIGINT | NOT NULL, CHECK `> 0` | 사용자 안에서 단조 증가하는 권한 버전 |
| `grant_status` | VARCHAR(16) | NOT NULL, CHECK | `active`, `revoked` |
| `claims_hash` | BYTEA | NOT NULL, CHECK 32 bytes | canonical claim snapshot hash |
| `source` | VARCHAR(32) | NOT NULL | 사용자 생성, 담당 Context event, 운영 승인 등 변경 출처 |
| `source_revision` | VARCHAR(128) | NOT NULL | 권한 원천의 버전 또는 event ID |
| `valid_from` | TIMESTAMPTZ | NOT NULL | 효력 시작 시각 |
| `valid_until` | TIMESTAMPTZ | NULL | 임시 권한 만료 시각 |
| `changed_by_user_id` | UUID | NULL | 운영 변경 주체 `user_id` |
| `change_reason` | VARCHAR(500) | NULL | 변경 사유 코드/설명 |
| `revoked_at` | TIMESTAMPTZ | NULL | 교체/회수 시각 |
| `updated_at` | TIMESTAMPTZ | NOT NULL | 최근 변경 시각 |
| `row_version` | BIGINT | NOT NULL, 기본 0 | 동시 갱신 방지 |

- 사용자별 active AccessGrant는 하나다. 교체할 때 기존 grant를 닫고 `grant_version`을 증가시킨다.
- CUSTOMER 기본 role은 Registration 완료 후 부여하고, SELLER/PLATFORM_OPERATOR는 담당 Context event 또는 승인된 운영 Command만 반영한다.
- `valid_until`이 있는 active grant는 만료 worker가 같은 transaction에서 revoked로 닫고 영향받는 Session을 폐기한다.
- `roles`와 `permissions`에는 빈 값과 중복을 허용하지 않는다. 애플리케이션이 정렬한 canonical 배열을 저장하고 `claims_hash`를 계산한다.
- access JWT는 `sub = user_id`, `sid = session_id`, roles, `permission_version = grant_version`, `iat`, `exp`, `jti` 같은 최소 claim만 사용한다. 이메일, 휴대폰, 표시명과 Identity 원문은 JWT나 `X-User-*` header에 넣지 않는다.
- 짧은 TTL access JWT 원문과 `jti` 목록은 저장하지 않는다. 즉시 폐기가 필요한 고위험 API는 Session의 `access_grant_version`과 사용자별 active `grant_version`을 DB 또는 캐시로 비교한다.

### `auth_reauthentication_proofs`

고위험 작업 직전의 최근 인증 결과를 role/permission `AccessGrant`와 분리해 저장한다. API에는 `proof_id.secret` 형식의 opaque 값을 반환하고 DB에는 secret의 keyed hash만 둔다.

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `reauthentication_proof_id` | UUID | PK | proof 공개 식별자 |
| `proof_hash` | BYTEA | UNIQUE, NOT NULL, CHECK 32 bytes | proof ID와 secret을 묶은 HMAC |
| `proof_key_version` | SMALLINT | NOT NULL | HMAC key 버전 |
| `user_id` | UUID | NOT NULL | proof 대상 사용자 |
| `session_id` | UUID | FK, NOT NULL | proof를 발급한 active Session |
| `authenticated_identity_id` | UUID | FK, NOT NULL | 강한 재인증에 사용한 Identity |
| `authentication_method` | VARCHAR(32) | NOT NULL, CHECK | MVP 재인증은 `email_password` |
| `purpose` | VARCHAR(64) | NOT NULL, CHECK | `replace_phone`, `link_identity`, `manual_recovery` 등 허용 작업 하나 |
| `authenticated_at` | TIMESTAMPTZ | NOT NULL | 비밀번호 또는 강한 인증을 다시 확인한 시각 |
| `expires_at` | TIMESTAMPTZ | NOT NULL | 기본 후보 5분의 절대 만료 시각 |
| `consumed_at` | TIMESTAMPTZ | NULL | 대상 작업이 proof를 소비한 시각 |
| `invalidated_at` | TIMESTAMPTZ | NULL | Session 폐기나 보안 사건으로 무효화한 시각 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 생성 시각 |

- 발급 시 Session이 active이고 `Session.user_id = proof.user_id`이며 authenticated Identity의 active Link도 같은 사용자여야 한다. `expires_at`은 Session의 idle/absolute 만료를 넘지 않는다.
- 소비 transaction은 proof row를 `FOR UPDATE`하고 hash, 목적, user, session, 만료, 미소비·미무효 상태를 모두 검증한 뒤 대상 IdentityLink 변경과 `consumed_at` 기록을 함께 commit한다.
- proof 원문, hash, HMAC key version은 log, trace, outbox, 감사 payload에 넣지 않는다.
- Session 폐기와 refresh 재사용 탐지는 해당 Session의 미소비 proof를 같은 transaction에서 `invalidated_at`으로 닫는다.

### `auth_idempotency_records`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `idempotency_record_id` | UUID | PK | IdempotencyRecord 식별자 |
| `operation` | VARCHAR(64) | NOT NULL | Command 또는 외부 event consumer 이름 |
| `scope_hash` | BYTEA | NOT NULL, CHECK 32 bytes | 사용자/Session/guest process 범위의 HMAC |
| `key_hash` | BYTEA | NOT NULL, CHECK 32 bytes | 원본 idempotency key의 HMAC |
| `request_hash` | BYTEA | NOT NULL, CHECK 32 bytes | secret을 포함한 전체 canonical request의 keyed HMAC fingerprint |
| `status` | VARCHAR(16) | NOT NULL, CHECK | `processing`, `completed`, `failed` |
| `resource_type` | VARCHAR(64) | NULL | 생성/변경된 Aggregate type |
| `resource_id` | UUID | NULL | 결과 Aggregate ID |
| `result_code` | VARCHAR(64) | NULL | 재시도 시 반환할 비민감 결과 코드 |
| `replay_payload_ref` | UUID | FK, NULL | 민감 응답 재생이 필요한 경우 전용 payload 참조 |
| `created_at` | TIMESTAMPTZ | NOT NULL | claim 시각 |
| `completed_at` | TIMESTAMPTZ | NULL | 처리 완료 시각 |
| `expires_at` | TIMESTAMPTZ | NOT NULL | 중복 방지 보존 만료 |

- `operation + scope_hash + key_hash`는 유일하다. 같은 범위의 key에 다른 `request_hash`가 들어오면 `IDEMPOTENCY_KEY_REUSED`로 거부한다.
- request fingerprint는 비밀번호, code, proof를 제거하지 않고 전체 canonical payload에 전용 서버 비밀키 HMAC을 적용한다. 원문이나 단순 hash를 저장하지 않아 offline verifier로 쓰이지 않게 한다.
- 회원가입 시작/완료, 이메일·휴대폰 로그인, 인증번호 재전송, 모바일 refresh, 번호 변경, 운영자 명령, 외부 사용자/권한 event 소비에 적용한다.
- IdempotencyRecord 본문에는 token, cookie, 인증번호, 암호화한 전체 응답을 저장하지 않는다. refresh만 전용 short-TTL replay payload reference를 둘 수 있다.
- 외부 event는 `operation = consumer_name`, `key_hash = HMAC(message_id)`로 기록하고 상태 변경과 `completed` 전환을 같은 트랜잭션에서 처리한다.

### `auth_idempotency_replay_payloads`

모바일 refresh의 같은 Idempotency-Key 재시도는 새 rotation을 만들지 않고 최초 token 묶음을 돌려줘야 한다. 이 예외를 일반 IdempotencyRecord나 SessionCredential에 넣지 않고 별도 보안 레코드로 격리한다.

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `replay_payload_id` | UUID | PK | IdempotencyRecord의 opaque reference |
| `payload_kind` | VARCHAR(32) | NOT NULL, CHECK | MVP는 `mobile_refresh_response` |
| `payload_ciphertext` | BYTEA | NULL | access JWT와 새 refresh token 응답의 envelope ciphertext |
| `payload_key_id` | VARCHAR(128) | NOT NULL | replay 전용 encryption key 참조 |
| `binding_hash` | BYTEA | NOT NULL, CHECK 32 bytes | operation, scope, key, request, Session, rotation 결과 binding |
| `replay_count` | SMALLINT | NOT NULL, 기본 0 | 동일 요청 재생 횟수 |
| `expires_at` | TIMESTAMPTZ | NOT NULL | API refresh 재시도 기간보다 길고 token TTL보다 짧은 만료 |
| `destroyed_at` | TIMESTAMPTZ | NULL | ciphertext 삭제/crypto-shred 시각 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 생성 시각 |

- IdempotencyRecord별 replay payload는 하나이며 `replay_payload_ref`는 refresh operation에서만 허용한다.
- 최초 refresh transaction은 credential rotation, encrypted replay payload, completed IdempotencyRecord, outbox를 함께 저장한다.
- 같은 scope/key/request hash의 재시도는 만료 전 ciphertext를 메모리에서만 복호화해 같은 응답을 반환하고 `replay_count`만 증가시킨다. 새 credential을 만들지 않는다.
- payload가 만료되면 ciphertext를 즉시 지우고 같은 key의 재시도에 `AUTH_REFRESH_RETRY_EXPIRED`를 반환한다. 다른 key로 rotated credential을 제시하면 refresh reuse 정책을 적용한다.
- replay ciphertext/key ID는 log, trace, event, 일반 cache에 넣지 않는다.

### `auth_outbox_events`

| 필드 | PostgreSQL 타입 | 제약 | 설명 |
| --- | --- | --- | --- |
| `event_id` | UUID | PK | OutboxEvent와 외부 event의 멱등 식별자 |
| `aggregate_type` | VARCHAR(64) | NOT NULL | Domain Aggregate type |
| `aggregate_id` | UUID | NOT NULL | 변경된 Aggregate ID |
| `aggregate_version` | BIGINT | NOT NULL | 변경 후 Aggregate version |
| `event_type` | VARCHAR(128) | NOT NULL | 의미가 구분된 Domain/Audit event type |
| `payload` | JSONB | NOT NULL | 버전이 명시된 비민감 event payload |
| `correlation_id` | UUID | NOT NULL | 요청 상관관계 ID |
| `causation_id` | UUID | NULL | 원인 Command/event ID |
| `occurred_at` | TIMESTAMPTZ | NOT NULL | 업무 사건 시각 |
| `publish_status` | VARCHAR(16) | NOT NULL, CHECK | `pending`, `published`, `dead_letter` |
| `publish_attempts` | INTEGER | NOT NULL, 기본 0 | 전송 시도 횟수 |
| `next_attempt_at` | TIMESTAMPTZ | NOT NULL | 다음 전송 가능 시각 |
| `published_at` | TIMESTAMPTZ | NULL | broker 확인 완료 시각 |
| `last_error_code` | VARCHAR(64) | NULL | 비민감 전송 실패 코드 |
| `created_at` | TIMESTAMPTZ | NOT NULL | 저장 시각 |

- `aggregate_type + aggregate_id + aggregate_version + event_type`는 유일하다.
- 상태 변경 Aggregate가 없는 미존재 계정 로그인 실패, synthetic reset, 재인증 실패 같은 시도 event는 `aggregate_type=AuthenticationAttempt`, `aggregate_id=attempt_id`, `aggregate_version=1`을 사용한다. `attempt_id`는 IdempotencyRecord 또는 서버 Command ID와 같고 같은-key 재생은 새 event를 만들지 않는다.
- payload에는 `user_id`, `identity_id`, `session_id`, 변경 사유 코드, 정책 버전 같은 참조값만 넣는다.
- 비밀번호/hash, 인증번호/hash, token/hash, 이메일·휴대폰 원문/ciphertext, provider secret, KMS key ID를 넣지 않는다.
- publisher는 `FOR UPDATE SKIP LOCKED`로 pending batch를 claim하고 at-least-once로 전송한다. 소비자는 `event_id`로 중복을 제거한다.
- 감사 broker 장애가 업무 트랜잭션을 롤백시키지 않도록 outbox insert까지만 업무 성공 조건으로 둔다. outbox insert가 실패하면 업무 트랜잭션도 실패한다.

## Aggregate 매핑

| 도메인 모델 | 저장 모델 | 매핑 방식 | 주의점 |
| --- | --- | --- | --- |
| `Identity` | `auth_identities` | Aggregate 1개를 1행으로 저장 | 인증값 ciphertext와 lookup hash는 도메인 밖 crypto adapter가 변환 |
| `PasswordCredential` | `auth_password_credentials` | Identity 하위 credential 이력을 N행으로 저장 | active 행은 1개, 교체된 verifier는 즉시 제거 |
| `VerificationChallenge` | `auth_verification_challenges` | challenge 1개를 1행으로 저장 | 검증·재전송 경합에서 row lock 사용 |
| 발송 인프라 | `auth_verification_delivery_payloads` | Challenge 발송 순번별 단기 ciphertext | Domain/Audit payload와 분리 |
| `IdentityLink` | `auth_identity_links` | 연결과 교체 이력을 append/update 혼합으로 저장 | active 유일성은 partial unique index가 최종 보장 |
| 정책 Value Object | `auth_policy_versions`와 하위 rule 테이블 | 활성 정책 전체를 불변 version snapshot으로 복원 | 일부 rule만 별도 version으로 섞지 않음 |
| `Registration` | `auth_registrations` | 외부 사용자 생성을 포함한 process 상태를 1행에 저장 | 외부 호출을 DB 트랜잭션 안에서 실행하지 않음 |
| `PasswordReset` | `auth_password_resets` | 소유 증명과 비밀번호 교체 상태를 1행에 저장 | 완료 transaction에서 전체 Session/refresh family 폐기 |
| `AuthenticationIntent` | `auth_authentication_intents` | 검증된 내부 복귀 의도를 1행에 저장 | opaque token은 hash만 저장하고 1회 소비 |
| ActionIntentPayload | `auth_action_intent_payloads` | action별 최소 payload를 단기 ciphertext로 저장 | 로그인 뒤 BFF 1회 전달 후 crypto-shred |
| `Session` | `auth_sessions` | session 1개를 1행으로 저장 | 상태와 절대 만료가 기준 |
| `SessionCredential` | `auth_session_credentials` | session 하위 credential과 회전 연결을 N행으로 저장 | active/rotated 상태를 삭제하지 않아 재사용 탐지 |
| `AccessGrant` | `auth_access_grants` | 사용자별 grant version을 N행으로 저장 | Session은 발급 당시 grant ID/version을 참조 |
| `UserAuthState` | `auth_user_auth_states` | user_id별 현재 restriction version을 1행으로 저장 | 새 Session/refresh/재인증의 fail-closed gate |
| `ReauthenticationProof` | `auth_reauthentication_proofs` | opaque proof의 hash와 목적·소비 상태를 1행으로 저장 | AccessGrant와 분리하고 Session 폐기 시 무효화 |
| `IdempotencyRecord` | `auth_idempotency_records` | Command/event key별 1행 | 민감 응답은 저장하지 않음 |
| 멱등 재생 인프라 | `auth_idempotency_replay_payloads` | refresh 최초 응답의 단기 ciphertext | IdempotencyRecord에는 reference만 저장 |
| `OutboxEvent` | `auth_outbox_events` | Aggregate 변경과 같은 transaction에서 append | Context 감사 원장이 아님 |

## 핵심 제약과 인덱스

다음 SQL은 구현 시 유지해야 할 핵심 제약의 형태를 보여준다. 실제 migration에서는 제약 이름과 허용 상태 값을 같은 migration 버전에서 확정한다.

```sql
CREATE UNIQUE INDEX uq_auth_policy_versions_active
    ON auth_policy_versions (status)
    WHERE status = 'active';

CREATE UNIQUE INDEX uq_auth_identities_value
    ON auth_identities (
        identity_type,
        identity_namespace,
        value_lookup_hash
    );

ALTER TABLE auth_identities
    ADD CONSTRAINT uq_auth_identities_id_type
    UNIQUE (identity_id, identity_type);

ALTER TABLE auth_identity_links
    ADD CONSTRAINT fk_auth_identity_links_identity_type
    FOREIGN KEY (identity_id, identity_type)
    REFERENCES auth_identities (identity_id, identity_type);

CREATE UNIQUE INDEX uq_auth_password_credentials_active
    ON auth_password_credentials (identity_id)
    WHERE status = 'active';

CREATE UNIQUE INDEX uq_auth_challenges_issued
    ON auth_verification_challenges (context_type, context_id, purpose)
    WHERE status = 'issued';

CREATE UNIQUE INDEX uq_auth_delivery_challenge_sequence
    ON auth_verification_delivery_payloads (
        challenge_id,
        send_sequence
    );

CREATE UNIQUE INDEX uq_auth_identity_links_identity_active
    ON auth_identity_links (identity_id)
    WHERE link_status = 'active';

CREATE UNIQUE INDEX uq_auth_identity_links_identity_requested
    ON auth_identity_links (identity_id)
    WHERE link_status = 'requested';

CREATE UNIQUE INDEX uq_auth_identity_links_user_primary_active
    ON auth_identity_links (user_id, identity_type)
    WHERE link_status = 'active'
      AND identity_type IN ('email', 'phone');

CREATE UNIQUE INDEX uq_auth_identity_links_previous
    ON auth_identity_links (previous_identity_link_id)
    WHERE previous_identity_link_id IS NOT NULL;

CREATE UNIQUE INDEX uq_auth_registrations_email_active
    ON auth_registrations (email_identity_id)
    WHERE status IN (
        'pending_verification',
        'requesting_user',
        'linking',
        'issuing_session'
    );

CREATE UNIQUE INDEX uq_auth_registrations_phone_active
    ON auth_registrations (phone_identity_id)
    WHERE status IN (
        'pending_verification',
        'requesting_user',
        'linking',
        'issuing_session'
    );

CREATE UNIQUE INDEX uq_auth_password_resets_active
    ON auth_password_resets (identity_id)
    WHERE identity_id IS NOT NULL
      AND status IN ('requested', 'challenge_verified');

CREATE UNIQUE INDEX uq_auth_session_credentials_session_active
    ON auth_session_credentials (session_id, credential_type)
    WHERE credential_status = 'active';

CREATE UNIQUE INDEX uq_auth_session_credentials_family_active
    ON auth_session_credentials (refresh_family_id)
    WHERE credential_status = 'active'
      AND refresh_family_id IS NOT NULL;

ALTER TABLE auth_session_credentials
    ADD CONSTRAINT fk_auth_session_credentials_rotated_from
        FOREIGN KEY (rotated_from_credential_id)
        REFERENCES auth_session_credentials (session_credential_id)
        DEFERRABLE INITIALLY DEFERRED,
    ADD CONSTRAINT fk_auth_session_credentials_rotated_to
        FOREIGN KEY (rotated_to_credential_id)
        REFERENCES auth_session_credentials (session_credential_id)
        DEFERRABLE INITIALLY DEFERRED;

CREATE UNIQUE INDEX uq_auth_access_grants_user_active
    ON auth_access_grants (user_id)
    WHERE grant_status = 'active';

CREATE UNIQUE INDEX uq_auth_access_grants_user_version
    ON auth_access_grants (user_id, grant_version);

ALTER TABLE auth_access_grants
    ADD CONSTRAINT uq_auth_access_grants_session_ref
    UNIQUE (access_grant_id, user_id, grant_version);

ALTER TABLE auth_identity_links
    ADD CONSTRAINT uq_auth_identity_links_session_ref
    UNIQUE (identity_link_id, identity_id, user_id);

ALTER TABLE auth_sessions
    ADD CONSTRAINT fk_auth_sessions_identity_link
    FOREIGN KEY (identity_link_id, identity_id, user_id)
    REFERENCES auth_identity_links (identity_link_id, identity_id, user_id),
    ADD CONSTRAINT fk_auth_sessions_access_grant
    FOREIGN KEY (access_grant_id, user_id, access_grant_version)
    REFERENCES auth_access_grants (
        access_grant_id,
        user_id,
        grant_version
    );

ALTER TABLE auth_sessions
    ADD CONSTRAINT uq_auth_sessions_id_user
    UNIQUE (session_id, user_id);

ALTER TABLE auth_reauthentication_proofs
    ADD CONSTRAINT fk_auth_reauthentication_proofs_session
    FOREIGN KEY (session_id, user_id)
    REFERENCES auth_sessions (session_id, user_id);

CREATE UNIQUE INDEX uq_auth_idempotency_operation_key
    ON auth_idempotency_records (operation, scope_hash, key_hash);

CREATE UNIQUE INDEX uq_auth_idempotency_replay_ref
    ON auth_idempotency_records (replay_payload_ref)
    WHERE replay_payload_ref IS NOT NULL;

CREATE UNIQUE INDEX uq_auth_outbox_aggregate_event
    ON auth_outbox_events (
        aggregate_type,
        aggregate_id,
        aggregate_version,
        event_type
    );
```

| 저장 모델 | 보조 인덱스 | 조회 목적 |
| --- | --- | --- |
| `auth_identities` | `idx_auth_identities_status_lock (credential_status, lock_until)` | 잠금 해제/운영 조회 batch |
| `auth_verification_challenges` | PK `challenge_id` | challenge를 잠근 뒤 token/SMS OTP hash 비교 |
| `auth_verification_challenges` | `idx_auth_challenges_expiry (status, expires_at)` | 만료 처리와 정리 |
| `auth_verification_delivery_payloads` | `idx_auth_delivery_pending (delivery_status, expires_at)` | relay 재시도와 crypto-shred |
| `auth_identity_links` | `idx_auth_identity_links_user_status (user_id, link_status)` | 사용자 인증 수단 목록 |
| `auth_identity_links` | `idx_auth_identity_links_previous (previous_identity_link_id)` | 번호 교체 계보 조회 |
| `auth_identity_links` | `idx_auth_identity_links_intent_expiry (intent_expires_at) WHERE link_status = 'requested'` | link/replacement intent 만료 처리 |
| `auth_registrations` | `idx_auth_registrations_status_updated (status, updated_at)` | 중단된 process 재처리 |
| `auth_password_resets` | `idx_auth_password_resets_status_expiry (status, expires_at)` | 만료 처리 |
| `auth_authentication_intents` | `idx_auth_intents_status_expiry (status, expires_at)` | active 조회와 정리 |
| `auth_action_intent_payloads` | `idx_auth_action_payload_expiry (expires_at) WHERE destroyed_at IS NULL` | 전달/만료 ciphertext 정리 |
| `auth_sessions` | `idx_auth_sessions_user_status (user_id, session_status)` | 전체 로그아웃/비밀번호 reset |
| `auth_sessions` | `idx_auth_sessions_expiry (session_status, absolute_expires_at)` | 만료 처리 |
| `auth_session_credentials` | `idx_auth_session_credentials_hash (secret_hash)` | cookie/refresh token 검증 |
| `auth_session_credentials` | `idx_auth_session_credentials_family (refresh_family_id, issued_at)` | 회전 계보와 family 폐기 |
| `auth_access_grants` | `idx_auth_access_grants_user_status (user_id, grant_status)` | 역할 변경 시 active grant 갱신 |
| `auth_user_auth_states` | PK `user_id`, UNIQUE `source_event_id` | 인증 gate 조회와 원천 event 멱등 반영 |
| `auth_reauthentication_proofs` | `idx_auth_reauthentication_proofs_expiry (expires_at) WHERE consumed_at IS NULL AND invalidated_at IS NULL` | 미소비 proof 검증과 만료 정리 |
| `auth_reauthentication_proofs` | `idx_auth_reauthentication_proofs_session (session_id) WHERE consumed_at IS NULL AND invalidated_at IS NULL` | Session 폐기 시 일괄 무효화 |
| `auth_idempotency_records` | `idx_auth_idempotency_expiry (expires_at)` | 보존 만료 정리 |
| `auth_idempotency_replay_payloads` | `idx_auth_replay_expiry (expires_at) WHERE destroyed_at IS NULL` | 만료 ciphertext 제거 |
| `auth_outbox_events` | `idx_auth_outbox_pending (next_attempt_at, created_at) WHERE publish_status = 'pending'` | publisher batch claim |

`expires_at > now()`처럼 현재 시각에 의존하는 조건은 partial index predicate에 넣지 않는다. PostgreSQL index predicate는 immutable 표현이어야 하므로 만료 worker와 읽기 쿼리가 상태를 명시적으로 전환한다.

## Repository 설계

Repository는 Aggregate 단위 인터페이스를 제공한다. 여러 Aggregate를 한 transaction에서 바꾸는 조정은 서비스의 Unit of Work가 담당하며, Repository가 임의로 commit하지 않는다.

| Repository | 주요 메서드 | 잠금/쿼리 기준 | 반환 또는 저장 대상 |
| --- | --- | --- | --- |
| `PolicyRepository` | `FindActive`, `Activate`, `FindByVersion` | active version과 모든 하위 rule | 불변 정책 snapshot |
| `IdentityRepository` | `FindByID`, `FindByValueHash`, `FindOrReserveUnownedForUpdate`, `FindByIDForUpdate`, `Save(expectedVersion)` | type + namespace + lookup hash, identity_id | Identity |
| `PasswordCredentialRepository` | `FindActiveByIdentityID`, `ReplaceActive`, `Revoke` | identity_id + active | PasswordCredential |
| `VerificationChallengeRepository` | `FindByIDForUpdate`, `FindIssuedByContextForUpdate`, `Save` | challenge_id 또는 context + purpose | VerificationChallenge |
| `VerificationDeliveryPayloadRepository` | `Insert`, `ClaimForRelay`, `MarkTerminalAndDestroy` | delivery payload ID, status + expiry | 암호화 발송 payload |
| `IdentityLinkRepository` | `FindActiveByIdentityID`, `FindRequestedIntentForUpdate`, `ListActiveByUserID`, `FindActivePrimaryForUpdate`, `Save` | identity_id, requested intent id, user_id + type | IdentityLink |
| `RegistrationRepository` | `FindByID`, `FindByIDForUpdate`, `FindByUserCreationRequestID`, `Save` | registration_id, request_id | Registration |
| `PasswordResetRepository` | `FindByIDForUpdate`, `FindActiveByIdentityID`, `Save` | reset_id, nullable email Identity | PasswordReset |
| `AuthenticationIntentRepository` | `Create`, `FindByOwnerProofHashForUpdate`, `Consume` | owner proof hash | AuthenticationIntent |
| `ActionIntentPayloadRepository` | `Insert`, `FindForDelivery`, `MarkDeliveredAndDestroy`, `DestroyExpired` | action payload ID + expiry | 암호화 action payload |
| `SessionRepository` | `FindByID`, `FindByIDForUpdate`, `ListActiveByUserIDForUpdate`, `Save` | session_id, user_id + active | Session |
| `SessionCredentialRepository` | `FindBySecretHashForUpdate`, `FindFamilyForUpdate`, `Insert`, `Save` | secret hash, refresh_family_id | SessionCredential |
| `AccessGrantRepository` | `FindActiveByUserID`, `FindByID`, `ReplaceActive` | user_id + active 또는 grant ID | AccessGrant |
| `UserAuthStateRepository` | `FindByUserID`, `FindByUserIDForUpdate`, `ApplyHigherVersion` | user_id, source event/version | UserAuthState |
| `ReauthenticationProofRepository` | `Insert`, `FindByHashForUpdate`, `Consume`, `InvalidateBySession` | proof hash, session_id + 미소비 | ReauthenticationProof 보안 레코드 |
| `IdempotencyRepository` | `Claim`, `Find`, `Complete` | operation + scope hash + key hash | IdempotencyRecord |
| `IdempotencyReplayPayloadRepository` | `Insert`, `FindValid`, `IncrementReplayCount`, `DestroyExpired` | replay payload ID + expiry | 암호화 refresh 응답 |
| `OutboxRepository` | `Append`, `ClaimPendingBatch`, `MarkPublished`, `ScheduleRetry` | pending + next_attempt_at | OutboxEvent |

### Repository 반환 규칙

- `FindByValueHash`, `FindByIDForUpdate`, `FindBySecretHashForUpdate`는 외부에 존재 여부를 그대로 노출하지 않는다. 서비스가 public error를 동일한 인증 실패 코드로 변환하고 내부 reason code는 감사 event로만 남긴다.
- Repository는 ciphertext를 자동으로 복호화한 도메인 객체를 범용 반환하지 않는다. 실제 발송이나 마스킹 조회처럼 허용된 use case만 crypto adapter를 호출한다.
- optimistic update는 `WHERE id = $1 AND row_version = $2`로 수행하고 성공 시 `row_version = row_version + 1`로 바꾼다. 영향 행이 0이면 동시 변경 오류로 처리한다.
- 목록 Repository에는 무제한 조회를 두지 않는다. CS 조회와 정리 batch는 cursor와 상한을 필수로 받는다.

## 쓰기 전략과 트랜잭션

### 공통 순서

1. API에서 Idempotency-Key가 필수인 명령은 `auth_idempotency_records`에서 operation/scope/key를 claim한다. 이메일·휴대폰 로그인, 회원가입 완료, 모바일 refresh는 Session 결과 참조를 반드시 기록한다. 내부 전용 일반 Session 발급만 이 단계를 생략할 수 있다.
2. Aggregate Root와 필요한 active 하위 행을 일정한 순서로 `FOR UPDATE` 잠근다.
3. 도메인 불변조건과 정책 버전을 확인하고 상태를 바꾼다.
4. Aggregate, IdempotencyRecord, OutboxEvent를 같은 transaction에 저장한다.
5. commit 후에만 cookie/token 원문을 응답하고 broker/외부 컨텍스트 호출은 별도 worker가 수행한다.

DB transaction 안에서 이메일/SMS provider, Context 사용자, Context 감사, broker를 호출하지 않는다.

### 회원가입 시작과 완료

| 단계 | 트랜잭션 경계 | 정합성 기준 | 실패 처리 |
| --- | --- | --- | --- |
| 가입 시작 | email/phone Identity 예약, PasswordCredential, Registration, idempotency claim | 정규화 값 유일성, active Registration별 Identity 예약 유일성, 비밀번호 hash 저장 | 귀속 Identity는 일반화된 가입 불가, 미귀속 expired/failed 예약은 cooldown 뒤 같은 행 재사용 |
| challenge 발급 | 상위 Registration/PasswordReset과 기존 issued Challenge row lock, 새 Challenge, delivery payload/outbox | API가 요청한 method 하나만 발급하고 같은 context/purpose의 issued Challenge는 하나 | 동일 key는 기존 결과, 새 재발급 key는 이전 Challenge를 revoked로 닫음 |
| challenge 확인 | challenge와 해당 Identity row lock, Registration 상태 갱신, 검증 event outbox | 만료 전 issued challenge만 1회 성공, 시도 횟수 원자 증가 | 제한 도달 시 failed로 닫고 감사 event 저장 |
| 사용자 생성 호출 준비 | Registration을 `pending_verification -> requesting_user`로 변경하고 고정 `user_creation_request_id` 저장 | 두 Challenge와 Identity 모두 verified | commit 뒤 Handler가 Context 사용자에 멱등 요청 |
| 사용자 생성 결과 반영 | Registration, 두 Identity/Link, UserAuthState row lock, 연결 outbox | 받은 `user_id`와 restriction version을 고정하고 active일 때만 `requesting_user -> linking -> issuing_session` | 제한 상태면 Link 결과를 보존하고 `failed(USER_AUTH_RESTRICTED)`, 유일성 충돌은 failed/manual operation |
| 자동 로그인 완료 | Registration, 두 Link, active AccessGrant, 새 Session/SessionCredential, AuthenticationIntent, 완료 IdempotencyRecord/outbox | email IdentityLink + `registration_verified`를 인증 근거로 `issuing_session -> completed`, Session 정확히 한 번 발급 | 응답 유실 시 저장된 `session_id`로 완료 여부를 확인하고 token 원문은 재생하지 않음 |

Context 사용자 생성은 분산 transaction이나 사용자 생성 요청 outbox로 묶지 않는다. Handler는 `requesting_user` commit 뒤 `UserContextPort.IssueUserId`를 호출하고, 응답을 별도 로컬 transaction에 저장한다. Context 사용자가 `user_id`를 만든 뒤 인증 연결이 일시 실패하면 같은 `user_creation_request_id`로 같은 응답을 다시 받아 재처리한다. Registration이 `issuing_session`까지 진행됐다면 Context 사용자를 다시 호출하지 않고 AccessGrant 조회와 Session 발급부터 재개한다. 재시도로 해결되지 않는 충돌을 사용자 계정 병합이나 자동 삭제로 보정하지 않는다.

로그인과 회원가입 완료 응답 유실 복구는 token 원문 저장이나 새 Session 생성으로 처리하지 않는다. completed IdempotencyRecord의 TTL 안에서 같은 key/request hash와 인증 proof를 확인하고, 현재 credential이 제시되지 않으면 결과가 가리키는 Session과 active SessionCredential을 잠근 뒤 기존 credential 폐기와 대체 credential 발급을 한 transaction으로 처리한다. 이메일 로그인은 비밀번호를 다시 검증하고, 휴대폰 로그인은 같은 challenge ID의 이전 verified 결과와 action key를 확인하며, 회원가입은 Registration owner proof를 확인한다. 현재 웹 cookie가 이미 유효하면 credential을 교체하지 않는다. 멱등 TTL이 끝나면 복구를 거부하고 새 로그인 action을 요구한다.

### 이메일 로그인과 잠금

1. email 정규화 값으로 lookup HMAC을 계산해 Identity와 active PasswordCredential을 읽고 Argon2id를 검증한다.
2. 검증 결과를 반영하는 transaction에서 Identity와 PasswordCredential의 현재 version을 다시 확인하고 Identity를 row lock한다.
3. 실패하면 현재 `login_failure_window_seconds` 안에서 `failure_count`를 원자적으로 증가시키고, 구간이 지났으면 `failure_window_started_at`과 count를 새로 시작한다. threshold에 도달하면 `locked`, `lock_until`, `lock_policy_version`을 함께 저장한다.
4. 비밀번호가 맞은 뒤 locked와 `password_reset_required`를 판정한다. 잠금 중이면 credential을 발급하지 않고, 재설정 필요면 강제 reset 결과만 반환한다. active일 때만 정책의 `reset_failure_on_success`에 따라 실패 구간과 만료된 잠금을 초기화하고, active AccessGrant를 참조하는 Session, SessionCredential, IdempotencyRecord, 로그인/세션 outbox를 함께 저장한다.
5. 비밀번호 hash의 `hash_parameters`가 오래되었으면 성공한 로그인 transaction에서 새 Argon2id parameter로 재hash하고 `credential_version`을 증가시킨다.

동시에 발생한 실패 요청은 Identity row lock으로 직렬화한다. 잠금 여부를 애플리케이션 메모리 카운터만으로 판단하지 않는다.

### 휴대폰 로그인

SMS challenge 확인 transaction에서 challenge와 기존 phone Identity를 함께 잠근다. code가 틀리고 실제 Identity가 있으면 LoginLockPolicy에 따라 failure count/window/lock을 원자적으로 갱신하되 외부 응답은 synthetic challenge와 같은 일반 실패로 유지한다. code가 맞으면 잠금 상태를 확인한 뒤 active IdentityLink와 사용자별 active AccessGrant를 확인한다. link가 없으면 Session을 만들지 않고 가입/연동 필요 코드를 반환한다. verified Identity와 active link가 확인되면 failure count를 초기화하고 grant ID/version을 참조하는 Session, SessionCredential, IdempotencyRecord, outbox를 같은 transaction에 저장한다.

### 비밀번호 재설정

1. 요청 단계에서는 verified 이메일 Identity와 active PasswordCredential을 찾는다. 이메일 credential 상태는 active, locked, password_reset_required를 허용하고 revoked/superseded는 제외한다. 휴대폰 방식은 verified phone Identity의 active Link로 같은 이메일 대상을 찾되 외부 응답으로 존재 여부를 구분하지 않는다.
2. PasswordReset만 만들고 존재 여부를 숨긴 접수 결과를 반환한다. 시작 transaction에서는 Challenge나 발송 outbox를 만들지 않는다.
3. 별도 challenge 발급 요청에서 PasswordReset을 잠그고 선택한 방식의 VerificationChallenge, delivery payload, 발송 outbox를 저장한다.
4. challenge 성공 시 1회용 reset grant를 발급해 hash만 저장하고 PasswordReset을 `challenge_verified`로 전환한다.
5. 완료 transaction은 PasswordReset, verified email Identity, active PasswordCredential, Identity owner의 active Session을 잠그고 reset grant를 constant-time으로 검증한다. email credential 상태는 active, locked, password_reset_required만 허용한다.
6. 이전 PasswordCredential을 `replaced`로 닫고 hash를 지운 뒤 새 credential version을 insert한다. Identity가 `password_reset_required`였다면 active로 되돌리고 required 시각/reason을 지운다. 해당 `user_id`의 전체 Session/SessionCredential과 refresh family를 폐기하고, 연결된 AuthenticationIntent를 `password_reset_completed`로 소비하며 reset/session 감사 outbox를 append한다. 사용자 권한 자체인 AccessGrant는 비밀번호 변경만으로 폐기하지 않는다.

### 재인증 proof 발급과 소비

1. 비밀번호 검증 뒤 발급 transaction은 UserAuthState active, verified email Identity의 `credential_status=active`, active PasswordCredential, 현재 Session/SessionCredential, 같은 `user_id`의 active email IdentityLink를 다시 확인하고 잠근다. 하나라도 아니면 proof를 발급하지 않는다. 통과하면 Session의 identity/link를 email Link로 재바인딩하고 `authentication_method=email_password`, `last_authenticated_at`을 갱신한다.
2. 기존 SessionCredential을 rotated/revoked로 닫고 채널에 맞는 대체 credential과 CSRF key version을 만든다. 동시에 목적 하나에 묶인 `auth_reauthentication_proofs` 행과 `auth.reauthentication.succeeded` 보안 감사 outbox를 저장한다.
3. opaque proof와 새 credential 원문은 commit 후 한 번만 반환하고 DB에는 keyed hash만 저장한다.
4. 인증 수단 연동이나 휴대폰 번호 교체 transaction은 proof를 `FOR UPDATE`하고 user, session, purpose, TTL, 미소비 상태를 검증한다.
5. 검증 성공 시 proof 소비와 대상 Aggregate 변경을 같은 transaction에서 처리한다. 대상 변경이 rollback되면 proof 소비도 rollback된다.
6. Session 폐기, refresh 재사용 탐지, 절대 만료 worker는 연결된 미소비 proof를 무효화한다.

비밀번호 불일치나 제한 초과도 `auth.reauthentication.failed` outbox를 남기되 user/session/purpose, 일반화된 reason만 포함한다. 비밀번호, proof 원문·hash, 이메일은 넣지 않는다.

### 휴대폰 번호 교체

이 작업은 기존 번호와 새 번호의 Identity를 identity_id 오름차순으로 잠근 뒤 다음 내용을 하나의 transaction에서 처리한다.

- 새 phone VerificationChallenge가 verified이고 현재 session의 대체 인증이 유효한지 확인한다.
- 기존 active phone IdentityLink를 `replaced`로 닫고 기존 phone Identity를 `superseded`로 바꾼다.
- 새 verified phone Identity를 같은 `user_id`에 active로 연결하고 `previous_identity_link_id`를 기록한다.
- 번호 교체 event와 감사 event를 outbox에 append한다.

active partial unique index 충돌은 계정 병합으로 해결하지 않고 `IDENTITY_ALREADY_LINKED`로 롤백한다.

### refresh rotation과 재사용 탐지

1. operation/scope/key를 claim한다. completed IdempotencyRecord의 request hash와 replay payload가 유효해도 현재 UserAuthState와 Session이 active인지 먼저 확인한다. 제한·폐기 상태면 payload를 파기하고 오류를 반환한다. 둘 다 active일 때만 최초 encrypted 응답을 복호화해 반환하고 rotation을 다시 실행하지 않는다.
2. 신규 요청이면 `secret_hash`로 SessionCredential을 `FOR UPDATE`하고 Session도 잠근다.
3. active이고 만료 전이면 기존 credential을 `rotated`로 바꾸고 새 active credential을 insert한다. 두 행의 rotation pointer, 새 token 묶음의 encrypted replay payload, completed IdempotencyRecord, outbox를 같은 transaction에서 저장한다.
4. 이미 `rotated` 또는 `revoked`인 credential이 다른 idempotency key로 제시되면 같은 refresh family를 잠그고 `revoke_family_and_session`을 적용한다.
5. reuse 상태, 관련 credential과 Session 폐기, 감사 outbox를 하나의 transaction에서 처리한다. 사용자 권한 자체인 AccessGrant는 폐기하지 않는다.
6. 서로 다른 refresh 요청이 동시에 들어오면 첫 transaction만 rotation에 성공한다. 같은 key는 그 결과를 재생하고 다른 key는 갱신된 상태에 따라 reuse 정책을 적용한다.

### 로그아웃과 권한 변경

- 현재 session 로그아웃은 Session과 active SessionCredential을 같은 transaction에서 폐기한다.
- 전체 로그아웃은 `user_id`의 active Session을 cursor batch로 잠그고 폐기한다. 한 transaction의 최대 행 수를 제한하고 operation ID로 재실행 가능하게 한다.
- 역할/권한 변경 event는 IdempotencyRecord로 중복을 제거하고, 해당 user의 active AccessGrant를 닫은 뒤 `grant_version + 1`인 snapshot을 만든다. 권한 축소/폐기 시 영향받는 Session을 SessionRevocationPolicy에 따라 폐기하고 cache invalidation outbox를 같은 transaction에 저장한다.
- 이미 발급된 짧은 TTL JWT는 만료 전까지 offline 서명 검증에 통과할 수 있다. 운영자 권한, 결제, 인증 수단 변경처럼 즉시 회수가 필요한 endpoint는 Session의 `access_grant_version`과 현재 active `grant_version`을 online 확인한다.

## 교착과 격리 수준

- 기본 격리 수준은 `READ COMMITTED`이고, 유일성은 partial unique index, 경합은 명시적 row lock으로 처리한다.
- 여러 Identity를 잠글 때는 `identity_id` 오름차순, 여러 Session을 잠글 때는 `session_id` 오름차순을 사용한다.
- 번호 교체, 사용자 생성 결과 반영처럼 여러 유일성 조건이 겹치는 transaction은 serialization failure와 deadlock을 제한 횟수만 재시도한다. 같은 Command ID와 request hash일 때만 재시도한다.
- 외부 API timeout은 DB transaction 재시도 조건이 아니다.
- Repository가 유니크 위반을 문자열로 해석하지 않도록 제약 이름을 안정적인 도메인 오류에 매핑한다.

## 암호화와 hash 전략

| 대상 | 저장 방식 | 키/회전 기준 | 비고 |
| --- | --- | --- | --- |
| 이메일·휴대폰·provider subject | AES-256-GCM envelope encryption + HMAC-SHA-256 lookup | encryption key는 KMS로 정기 회전, lookup key는 별도 관리 | ciphertext와 HMAC을 서로 다른 키로 보호 |
| 비밀번호 | salt와 parameter가 포함된 Argon2id PHC string | `hash_parameters` 정책으로 점진 재hash | reversible encryption 금지 |
| SMS OTP | server pepper로 `challenge_id + OTP`를 HMAC-SHA-256 | 짧은 TTL, 횟수 제한, constant-time 비교 | 6자리 값은 단순 SHA-256 금지 |
| 이메일 확인 token | `challenge_id + 256-bit CSPRNG token`의 HMAC-SHA-256 | token key version 저장 | challenge ID와 원문 token을 1회 전달 |
| cookie/refresh token | 256-bit 이상 CSPRNG credential의 HMAC-SHA-256 | token key version 저장 | access JWT와 분리 |
| 웹 Session CSRF token | SessionCredential ID와 `secret_hash`의 HMAC-SHA-256 도출값 | `csrf_key_version` 저장, credential 회전 시 함께 교체 | 원문/hash 별도 저장 없이 Session credential에 바인딩 |
| ReauthenticationProof | `proof_id + 256-bit CSPRNG secret`의 HMAC-SHA-256 | proof key version 저장, 기본 후보 TTL 5분 | user, Session, 목적에 바인딩하고 1회 소비 |
| verification delivery payload | 전용 DEK의 AES-256-GCM envelope encryption | Challenge TTL 이내, terminal 처리 후 crypto-shred | destination과 code/link만 포함 |
| action intent payload | 전용 DEK의 AES-256-GCM envelope encryption | AuthenticationIntent TTL 이내, 전달/만료 후 crypto-shred | action allowlist의 최소 필드만 포함 |
| refresh replay payload | 전용 DEK의 AES-256-GCM envelope encryption | 짧은 API retry TTL, 만료 즉시 crypto-shred | 최초 access/refresh 응답만 포함 |
| idempotency key | HMAC-SHA-256 | 전용 key 사용 | 외부 key 원문 미보관 |
| idempotency request fingerprint | 전체 canonical payload의 HMAC-SHA-256 | key fingerprint와 분리한 전용 key 사용 | password/code/proof 변경을 구분하되 원문·단순 hash는 미보관 |
| AccessGrant snapshot | canonical 배열의 SHA-256 | secret 불필요 | 무결성 비교용, 인증 verifier가 아님 |

### 정규화

- 이메일은 앞뒤 공백과 제어 문자를 제거하고 도메인 부분을 IDNA 규칙으로 정규화한다. provider별 dot/plus 변형을 임의로 같은 주소로 합치지 않는다.
- 휴대폰 번호는 국가 코드를 포함한 E.164 canonical 값으로 저장 전 변환한다.
- provider subject는 `provider namespace + issuer + subject` 조합을 정규화한다.
- 정규화 규칙 버전이 바뀌면 기존 값을 한 번에 덮지 않는다. dual-read와 충돌 보고서를 먼저 적용한다.

lookup HMAC key 회전은 encryption key 회전과 분리한다. HMAC key를 바꾸면 유일성 값도 바뀌므로 새 key dual-write, 전체 backfill, 중복 검증, read 전환, 이전 key 제거 순서로 별도 migration을 수행한다.

## 읽기 전략

| 화면/API 또는 내부 작업 | 조회 기준 | 사용 인덱스 | 캐시 |
| --- | --- | --- | --- |
| 이메일 로그인 | type + namespace + lookup hash, active PasswordCredential | `uq_auth_identities_value`, `uq_auth_password_credentials_active` | 사용하지 않음 |
| 휴대폰 로그인 | phone lookup hash, verified Identity, active link | identity value unique, identity active link | 사용하지 않음 |
| challenge 확인 | challenge_id + issued + expires_at, verifier hash constant-time 비교 | PK | 사용하지 않음 |
| 회원가입 상태 | registration_id | PK | 짧은 상태 poll cache 가능, DB가 기준 |
| 비밀번호 reset 상태 | password_reset_id | PK | 사용하지 않음 |
| 로그인 의도 소비 | owner proof hash + active + expires_at, 웹은 CSRF hash 추가 | owner proof unique | 사용하지 않음 |
| 로그인 뒤 action 복구 | consumed Intent의 action payload reference | payload PK + expiry | BFF 전달 중에만 메모리 복호화, cache 금지 |
| 재인증 proof 소비 | proof hash + user_id + session_id + purpose + 미소비 + expires_at | proof hash unique, session partial index | 사용하지 않음 |
| 웹 session 확인/CSRF 검증 | cookie secret hash -> SessionCredential, 저장된 CSRF key version으로 token 재도출 | credential secret unique | 검증 성공 후 session 최소 상태만 짧게 cache 가능 |
| 모바일 refresh | credential secret hash -> Session | credential secret unique | 검증 성공 후 session 최소 상태만 짧게 cache 가능 |
| access token online 확인 | session_id의 grant version과 user별 active grant version 비교 | Session PK, user active grant unique | Redis cache 허용 |
| 사용자 인증 상태 | user_id의 active link/session/grant | user/status 보조 인덱스 | CS 조회는 cache하지 않음 |
| 잠금 상태 운영 조회 | credential_status + lock_until | `idx_auth_identities_status_lock` | 사용하지 않음 |
| outbox 발행 | pending + next_attempt_at | `idx_auth_outbox_pending` | 사용하지 않음 |

### 캐시 경계

- session/grant cache에는 `user_id`, `session_id`, 상태, 역할/권한 snapshot, grant version, idle/absolute 만료 시각만 둔다.
- 이메일/휴대폰 ciphertext 또는 hash, 비밀번호 hash, challenge verifier, refresh token hash는 Redis에 복제하지 않는다.
- cache TTL은 DB의 `idle_expires_at`과 `absolute_expires_at` 중 먼저 오는 시각을 넘을 수 없다. 로그아웃, 재사용 탐지, 권한 변경 transaction은 commit된 outbox로 cache invalidation을 전송한다.
- cache miss는 DB 조회로 보완하고 cache hit만으로 만료·폐기를 연장하지 않는다.
- DB 장애 시 공개 탐색은 인증 DB를 호출하지 않는다. 일반 endpoint는 유효한 짧은 TTL JWT를 제한적으로 offline 검증할 수 있지만 token 발급, refresh, 인증 수단 변경, 운영자 작업은 fail closed한다.

## Outbox와 멱등성

### event 전달

- Aggregate 상태와 OutboxEvent insert가 같은 transaction에 있어야 한다.
- publisher는 성공한 broker acknowledgement 뒤에만 `published`로 바꾼다. publish 후 상태 갱신 전에 장애가 나면 같은 `event_id`가 다시 전달될 수 있다.
- Context 감사와 Context 사용자는 `event_id`를 consumer idempotency key로 사용한다.
- `dead_letter` 전환은 자동 삭제가 아니다. paging/alert 대상이며 운영자가 원인을 확인한 뒤 재시도한다.
- 회원가입, 로그인 성공/실패, challenge 성공/실패, 잠금, session 발급/갱신/로그아웃, refresh reuse, 비밀번호 reset, 인증 수단 연결/교체, 권한 변경을 서로 다른 `event_type`으로 둔다.

### Command 멱등성

- public API의 Idempotency-Key는 원문을 저장하지 않는다. operation namespace와 인증된 `user_id`/Session 또는 서버가 발급한 guest process scope를 분리해 HMAC한다.
- 같은 key와 같은 request hash의 완료 요청은 resource ID와 결과 코드로 현재 결과를 재구성한다. 모바일 refresh는 유효한 `replay_payload_ref`의 최초 암호화 응답을 재생한다.
- 같은 key와 다른 request hash는 거부한다.
- 로그인 실패가 completed인 IdempotencyRecord는 같은 key/request fingerprint에 같은 실패를 반환하고 비밀번호/challenge를 다시 검증하거나 failure count를 증가시키지 않는다. 성공 record의 credential 전달 복구만 인증 proof를 다시 확인한다.
- `processing` 레코드가 timeout을 넘겼다고 바로 새 작업을 만들지 않는다. 연결된 Aggregate/outbox 존재 여부를 확인한 repair worker만 재개한다.
- login과 challenge 검증의 실패 횟수는 idempotency key가 없는 반복 요청마다 실제 시도로 센다.
- refresh는 같은 scope/key/request hash에 한해 전용 encrypted replay payload로 최초 응답을 재생한다. payload가 만료되면 재발급하거나 rotation을 추가하지 않고 재인증을 요구한다.

## 마이그레이션

### 생성 순서

1. `auth_policy_versions`와 두 정책 rule 테이블을 만들고 요구사항에 확정된 access 15분, refresh 14일, remember-me 30일, rotation 사용, 로그인 실패 5회를 seed한다. 잠금 구간/시간, 웹 idle/absolute TTL, 목적별 인증 제한은 운영 승인값을 migration 입력으로 확정한다.
2. `auth_identities`와 `auth_password_credentials`를 만든다.
3. `auth_verification_challenges`, `auth_verification_delivery_payloads`, `auth_action_intent_payloads`, `auth_authentication_intents`를 만든다. 다형 `context_id`에는 DB FK를 만들지 않고 context type/ID를 서비스에서 검증한다.
4. `auth_identity_links`와 사용자별 `auth_access_grants`를 만든다.
5. `auth_user_auth_states`, `auth_sessions`, `auth_session_credentials`, `auth_reauthentication_proofs`를 만든 뒤 AuthenticationIntent/Proof의 Session FK와 IdentityLink의 `reauthentication_proof_id` FK를 연결한다.
6. `auth_registrations`와 `auth_password_resets`를 만들고 Challenge/Session 참조 FK를 연결한다.
7. `auth_idempotency_replay_payloads`, `auth_idempotency_records`, `auth_outbox_events`를 만들고 replay reference FK를 후속 `ALTER TABLE`로 연결한다.
8. CHECK/FK를 `NOT VALID`로 추가한 뒤 검증하고, 큰 보조 인덱스는 `CREATE INDEX CONCURRENTLY`로 생성한다.

### 기존 데이터 전환

- 기존 이메일/휴대폰을 canonical 값으로 변환하고 ciphertext와 lookup HMAC을 backfill한다. 유일성 인덱스를 만들기 전에 정규화 충돌 보고서를 생성하며, 충돌을 user 계정 병합으로 해결하지 않는다.
- 기존 active Link로 `owner_user_id`를 backfill한다. 같은 Identity의 과거 Link가 서로 다른 `user_id`를 가리키면 자동 이관을 중단하고 수동 검토 대상으로 분리한다.
- 기존 비밀번호 hash는 알고리즘과 `hash_parameters`를 표시한다. 안전한 verifier는 로그인 성공 시 Argon2id로 점진 교체하고, 안전하지 않은 형식은 Identity를 `password_reset_required`와 reason으로 표시한다.
- 기존 refresh token 원문이 있으면 신규 DB로 복사하지 않는다. cutover에서 기존 session을 만료시키거나, 이미 hash만 있는 안전한 credential만 명시적인 호환 기간 동안 이관한다.
- 기존 role 값은 canonical role 사전에 맞춰 AccessGrant snapshot으로 backfill한다. 알 수 없는 값은 기본 권한으로 조용히 치환하지 않고 migration을 중단한다.
- 기존 active Link의 `user_id`별 UserAuthState는 Context 사용자 restriction snapshot과 version을 받아 backfill한다. 확인되지 않은 사용자를 임의 active로 두지 않으며 로그인/refresh cutover 전에 누락 0건을 검증한다.
- outbox dual-write를 먼저 켠 뒤 consumer 중복 제거를 확인하고 기존 직접 publish 경로를 제거한다.

### 배포와 rollback

- expand migration -> 호환 코드 배포 -> backfill -> 검증/인덱스 활성화 -> 기존 경로 제거 순서를 따른다.
- migration은 forward-only다. rollback은 애플리케이션 feature flag와 이전 read path로 수행하며 새 인증 데이터를 삭제하는 down migration을 실행하지 않는다.
- crypto key rotation과 schema migration을 같은 배포에서 동시에 수행하지 않는다.

## 보존과 정리

보존 기간은 개인정보/감사 정책 확정 후 운영 설정으로 조정한다. 아래 값은 초기 운영 기준이며, Context 감사의 법적 보존 기간을 대신하지 않는다.

| 저장 모델 | 초기 보존 기준 | 정리 방식 |
| --- | --- | --- |
| `auth_identities` | owner/Link가 있으면 계정 생명주기 동안 보존, 미귀속 예약은 failed/expired 가입 뒤 cooldown 동안 보존 | 미귀속 예약은 같은 행 재예약 또는 승인된 기간 후 hard delete, 귀속 Identity는 ciphertext crypto-shred/lookup tombstone 정책 별도 승인 |
| `auth_password_credentials` | active만 verifier 보존 | 교체 transaction에서 이전 hash 즉시 NULL 처리, 메타데이터 90일 후 삭제 가능 |
| `auth_verification_challenges` | verified/failed/expired/revoked 후 30일 | hash와 행 삭제, 의미별 감사 event는 Context 감사에 보존 |
| `auth_verification_delivery_payloads` | terminal/만료 즉시 ciphertext 제거, 메타데이터 30일 | relay transaction에서 crypto-shred, 누락분은 만료 worker 처리 |
| `auth_identity_links` | active/closed 귀속 Link는 계정 추적 기간, 미완료 requested Link는 intent TTL 뒤 비민감 메타데이터 30일 | requested 만료 시 revoked 처리 후 미귀속 Identity ciphertext 정리, 귀속 Link는 hard delete보다 닫힘 상태 보존 |
| `auth_registrations` | completed 90일, failed/expired 30일 | `user_id`와 process 메타데이터 삭제, 감사 event 별도 보존 |
| `auth_password_resets` | 완료/만료 후 30일 | process 행 삭제, password verifier나 증명값 없음 |
| `auth_authentication_intents` | 소비/만료 후 24시간 | batch hard delete |
| `auth_action_intent_payloads` | BFF 전달 확인 또는 Intent 만료 즉시 ciphertext 제거, 메타데이터 24시간 | crypto-shred 뒤 batch hard delete |
| `auth_sessions` | 만료/폐기 후 90일 | session 메타데이터 삭제 전 credential 정리 |
| `auth_session_credentials` | 만료/폐기/reuse 탐지 후 90일 | secret hash 삭제, reuse 감사 event는 별도 보존 |
| `auth_access_grants` | revoked 또는 `valid_until` 종료 후 90일 이상 | grant snapshot 삭제 전 참조 Session 보존 여부 확인 |
| `auth_user_auth_states` | 사용자 계정 생명주기 동안 현재 상태 보존 | Context 사용자 삭제/익명화 정책과 조정, source event/reason은 감사 원장으로 이관 |
| `auth_reauthentication_proofs` | 소비·무효·만료 후 24시간 | proof hash를 제거한 뒤 비민감 상태 메타데이터 삭제 |
| `auth_idempotency_records` | public Command 24시간 이상, 외부 event 90일 | operation별 TTL batch delete |
| `auth_idempotency_replay_payloads` | refresh 재시도 TTL 종료 즉시 ciphertext 제거, 메타데이터 24시간 | 전용 만료 worker와 KMS key 폐기 |
| `auth_outbox_events` | published 후 7일 | published만 batch delete, pending/dead-letter 자동 삭제 금지 |
| `auth_policy_versions`와 하위 rules | 영구 보존 | 민감값이 없는 판정 근거이므로 삭제하지 않음 |

정리 worker는 작은 batch와 `SKIP LOCKED`를 사용한다. 대량 `DELETE` 후 vacuum 부하를 줄이기 위해 실행량과 주기를 제한하고, 삭제 건수와 가장 오래된 미처리 행 시각을 metric으로 남긴다.

## 운영과 관측

- DB metric: login query latency, row lock wait, unique violation별 횟수, challenge attempt update latency, refresh rotation conflict, outbox oldest pending age, dead-letter count를 수집한다.
- application metric: signin/signup/refresh/phone verification p50/p95/p99, 인증 실패율, 잠금률, refresh reuse 탐지율을 수집하되 identity 값은 label로 사용하지 않는다.
- log에는 `request_id`, `correlation_id`, Aggregate ID, error code만 넣고 token/hash/ciphertext/KMS key ID를 넣지 않는다.
- CS 조회는 exact email/phone 입력을 서버에서 HMAC한 exact match만 허용한다. 부분 검색이나 전체 ciphertext 복호화 목록 API를 만들지 않는다.
- 운영자의 잠금 해제, 인증 수단 수동 변경, 정책 변경은 일반 Repository 직접 호출이 아니라 인가된 Command, IdempotencyRecord, OutboxEvent를 거쳐야 한다.
- 백업은 encryption at rest와 접근 감사를 적용한다. KMS key와 DB backup을 같은 보안 경계에 보관하지 않는다.

## 수용 기준 추적

| 요구사항 | 영속성 보장 |
| --- | --- |
| 이메일·휴대폰을 모두 확인한 회원가입 | Registration이 두 verified Identity를 확인하고 같은 `user_id`에 link한 뒤 완료 |
| 이메일/SMS secret 안전한 발송 | Challenge verifier hash와 별도 short-TTL delivery ciphertext, terminal crypto-shred |
| 인증 식별자 단일 연결, 계정 병합 금지 | identity active link partial unique + 외부 `user_id` 간 자동 이동 금지 |
| 휴대폰 번호 교체 이력 | 기존 Identity/IdentityLink 닫힘 상태, 새 link, previous link, outbox를 한 transaction에 저장 |
| 로그인 실패 5회와 설정 가능한 잠금 | immutable policy version + Identity row lock + failure_count/lock_until |
| 비밀번호 재설정 | 단일 사용 reset grant, PasswordCredential 교체, 전체 Session/refresh family 폐기를 한 transaction에 저장 |
| access/refresh/remember-me TTL과 rotation | policy snapshot, Session 만료, SessionCredential family/rotation chain |
| refresh token 재사용 탐지 | rotated credential 보존 + family/session 원자 폐기 |
| refresh 멱등 재시도 | 같은 scope/key/request에는 최초 encrypted replay payload 반환, 다른 key 재사용은 보안 사건 처리 |
| 역할/권한 claim | 사용자별 versioned AccessGrant와 Session 발급 snapshot, email/profile 없는 최소 claim |
| 사용자 차단·비활성 | user_id별 UserAuthState restriction version, active Grant와 전체 Session 원자 폐기, 재로그인/refresh fail closed |
| 의미별 감사 event | Aggregate 변경과 같은 transaction의 민감정보 없는 OutboxEvent |
| 인증 서비스 책임 최소화 | 외부 `user_id`만 참조하며 이름/주소/프로필/약관 데이터를 저장하지 않음 |

## 확인 필요

- 첫 활성 정책의 실패 집계 구간, `lock_duration_seconds`, 웹 idle/absolute TTL을 운영 기준으로 확정해야 한다.
- SMS/email challenge TTL, 최대 시도 횟수, 재전송 제한의 초기값을 확정해야 한다.
- 닫힌 email/phone Identity의 ciphertext와 lookup tombstone 보존 기간을 개인정보 정책과 번호 재할당 정책에 맞춰 확정해야 한다.
- AccessGrant role/permission의 원천 컨텍스트, canonical 이름, 즉시 online 검증이 필요한 endpoint 목록을 확정해야 한다.
- MVP 이후 같은 `user_id`에 같은 type의 Identity를 여러 개 허용할 경우 `uq_auth_identity_links_user_primary_active`를 slot 기반 제약으로 확장해야 한다.
