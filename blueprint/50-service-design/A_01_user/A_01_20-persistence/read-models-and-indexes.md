---
id: SD.A.0120.READ-MODELS
title: Context 사용자 조회 모델과 인덱스
type: service-design-persistence
status: draft
tags: [service-design, user, read-model, index, cache]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 조회 모델과 인덱스

## 조회 모델

| 조회 | 원천 | 반환 | 캐시 |
| --- | --- | --- | --- |
| 본인 프로필 | user_accounts + user_profiles | private name, nickname, introduction, media URL, versions | 사용자 서비스 cache 없음 |
| 마이 사용자 조각 | user_accounts + user_profiles | account/profile 필드만 | BFF가 짧은 TTL을 선택할 수 있음 |
| 내부 계정 상태 | user_accounts | auth status, restriction version, account version | 소비자는 versioned event projection 우선 |
| 운영 상태 이력 | user_accounts + user_account_status_history | 상태 전이와 승인 참조 | cache 금지 |
| 가입 진행 상태 | user_provisionings | link 상태와 일반화 failure code | 운영·reconciliation 전용 |

사용자 서비스는 등급·주문·배송·쿠폰·포인트·찜·알림 section을 조회하거나 저장하지 않는다. BFF가 각 원천을 병렬 조회한다.

## 인덱스

| 테이블 | 인덱스 | 목적 |
| --- | --- | --- |
| user_registration_drafts | PK profile_request_id | 가입 초안 조회 |
| user_registration_drafts | `(status, expires_at)` | 만료 Worker |
| user_provisionings | UNIQUE source_event_id | Event 중복 제거 보조 |
| user_provisionings | UNIQUE auth_registration_id | Registration당 한 사용자 |
| user_provisionings | UNIQUE verification_binding_id | binding 재사용 방지 |
| user_provisionings | UNIQUE link_request_id | 성공·거부 Auth link 업무 멱등성 |
| user_provisionings | `(link_status, link_accept_until)` partial nonterminal | 기한 도래 reconciliation |
| user_provisionings | `(link_status, updated_at)` partial nonterminal | 처리 정체 age 탐지 |
| user_accounts | PK user_id | 계정 조회 |
| user_accounts | `(auth_status, restriction_version)` | 운영 집계·reconciliation |
| user_profiles | PK user_id | 프로필 조회 |
| user_account_status_history | UNIQUE `(user_id, restriction_version)` | 상태 변경 순서 |
| user_account_status_history | `(user_id, effective_at DESC)` | 사용자별 이력 |
| user_inbox_events | `(status, next_attempt_at, lease_until)` partial ready | retry/recovery worker |
| user_outbox_events | `(status, next_attempt_at, lease_until)` partial ready | relay/recovery worker |

닉네임 검색·유일성 인덱스는 정책과 실제 기능이 확정되기 전 만들지 않는다.

## MyPage User Slice

```json
{
  "userId": "00000000-0000-0000-0000-000000000001",
  "accountStatus": "active",
  "restrictionVersion": 1,
  "accountVersion": 3,
  "profile": {
    "nickname": "dropfan",
    "displayName": "dropfan",
    "introduction": "한정 드롭을 기다리는 중",
    "profileImage": {
      "available": true,
      "url": "https://media.example/signed/...",
      "expiresAt": "2026-07-10T10:10:00Z",
      "errorCode": null
    },
    "profileVersion": 4
  }
}
```

private name은 별도 본인 프로필 응답에서만 제공한다. 마이 화면에 필요하지 않으면 BFF slice에 포함하지 않는다.

## 이미지 URL

- DB에는 `profile_media_asset_id`만 저장한다.
- signed URL은 조회 시 미디어 Port에서 생성한다.
- signed URL과 함께 `expiresAt`을 반환하고 URL이 든 응답은 `Cache-Control: private, no-store`를 사용한다.
- 미디어 timeout은 프로필 전체를 가짜 성공으로 바꾸지 않는다. `profileImage.available=false`와 안정적인 오류 코드를 반환한다.
- signed URL을 Redis나 PostgreSQL에 장기 저장하지 않는다.

## 캐시와 정합성

- 사용자 서비스 내부 profile cache는 MVP에 두지 않는다. PK 조회와 인덱스로 시작한다.
- BFF cache를 사용할 경우 key에 `user_id`, `account_version`, `profile_version`을 반영하고 짧은 TTL과 명시적 `asOf`를 둔다. TTL은 signed URL의 `expiresAt`을 넘지 않는다.
- 계정 제한 Event가 발생하면 BFF user slice cache invalidation을 발행할 수 있다. 캐시 무효화 실패가 Auth 제한 적용을 지연시키면 안 된다.
- Auth와 권한 게이트는 `user.account-auth-state.v1`의 `User.AccountAuthStateChanged` version projection을 사용하고 사용자 API cache를 신뢰하지 않는다.

## 보존과 Migration

- 가입 초안: 만료·거부 뒤 정책 기간 보존 후 private name을 crypto-shred한다.
- 성공적으로 소비된 초안: 기본 UserProfile 생성과 Auth 연동 결과가 안정화되면 draft의 `private_name_ciphertext`를 즉시 crypto-shred하고 `profile_request_id`, registration process, 상태, Provisioning 귀속 같은 멱등 tombstone만 보존한다. 최종 private name은 UserProfile 원장 하나에만 남긴다.
- provisioning_failed: 장애 조사·중복 방지 기간 동안 식별자와 일반화 failure를 보존하고 프로필 원문은 먼저 제거한다.
- 상태 이력: 감사·분쟁 정책에 따라 append-only 보존한다.
- active 프로필: 탈퇴·개인정보 정책 확정 전 자동 삭제 migration을 만들지 않는다.
- migration은 expand -> backfill -> 검증 -> contract 순서로 수행하며 server 시작 시 자동 실행하지 않는다.

## 확인 필요

- BFF cache TTL과 invalidation event
- 미디어 signed URL timeout/TTL
- 가입 초안, 실패 Provisioning, 상태 이력의 보존 기간
- 탈퇴 후 UserId tombstone과 프로필 익명화 방식
