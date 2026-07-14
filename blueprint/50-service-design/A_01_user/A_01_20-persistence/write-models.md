---
id: SD.A.0120.WRITE
title: Context 사용자 쓰기 모델
type: service-design-persistence
status: draft
tags: [service-design, user, postgresql, write-model]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 쓰기 모델

## users

| 필드 | 제약 |
| --- | --- |
| `user_id` | UUID PK |
| `registration_id` | VARCHAR, UNIQUE, NOT NULL |
| `account_status` | `active`, `restricted`, `deactivated` |
| `private_name` | 암호화 저장, NOT NULL |
| `nickname` | 정규화 값, NOT NULL |
| `introduction` | NULL 허용 |
| `profile_media_asset_id` | 불투명 ID, NULL 허용 |
| `user_version` | BIGINT, NOT NULL, 1 이상 |
| `created_at`, `updated_at` | TIMESTAMPTZ |

계정 상태와 프로필을 별도 테이블이나 version으로 나누지 않는다.

## user_agreement_acceptances

| 필드 | 제약 |
| --- | --- |
| `user_id` | FK, NOT NULL |
| `agreement_code` | VARCHAR, NOT NULL |
| `agreement_version` | VARCHAR, NOT NULL |
| `accepted_at` | TIMESTAMPTZ, NOT NULL |

PK는 `(user_id, agreement_code, agreement_version)`이다. 가입 생성 트랜잭션에서만 기록하며 수정하지 않는다.

## user_status_history

| 필드 | 제약 |
| --- | --- |
| `status_change_id` | UUID PK |
| `user_id` | FK, NOT NULL |
| `previous_status`, `changed_status` | 허용 상태 값 |
| `reason_code` | 공개 가능한 코드 |
| `changed_by` | 운영 주체의 불투명 ID |
| `changed_at` | TIMESTAMPTZ |

상세 메모와 운영자 개인정보는 별도 감사 시스템에 두고 일반 User 원장에 복제하지 않는다.

## user_idempotency_records

| 필드 | 역할 |
| --- | --- |
| `operation`, `scope_id`, `idempotency_key` | UNIQUE 업무 범위 |
| `request_hash` | 같은 key의 다른 요청 탐지 |
| `result_type`, `result_id`, `result_version` | 결과 재구성 |
| `created_at`, `expires_at` | 보존 기한 |

응답 본문, 프로필 원문, proof와 URL은 저장하지 않는다.

## 조건부 UPDATE

프로필·이미지·상태 변경은 별도 잠금 조회 대신 다음 조건을 한 UPDATE에 포함한다.

```sql
UPDATE users
SET ..., user_version = user_version + 1, updated_at = now()
WHERE user_id = :user_id
  AND user_version = :expected_user_version
  AND account_status = 'active'
RETURNING user_id, user_version, updated_at;
```

상태 변경은 `account_status='active'` 대신 현재 상태와 허용 전이를 조건으로 사용한다.
