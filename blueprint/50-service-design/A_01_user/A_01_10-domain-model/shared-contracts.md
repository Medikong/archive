---
id: SD.A.0110.CONTRACTS
title: Context 사용자 공통 계약
type: service-design-domain-contracts
status: draft
tags: [service-design, user, contracts, event, policy]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
bounded_context: BC.A.01
---

# Context 사용자 공통 계약

## 식별자와 상태

| 이름 | 표현 | 규칙 |
| --- | --- | --- |
| UserId | UUID, JSON string | Context 사용자가 생성하며 변경·재사용·병합하지 않는다. |
| ProfileRequestId | opaque string | 가입 전 프로필 초안 참조다. |
| RegistrationProcessId | opaque string | BFF가 초안·동의·Auth Intent를 같은 가입 작업으로 묶는 사전 인증 scope다. |
| ProvisioningId | opaque string | 가입 장기 작업 식별자다. |
| LinkRequestId | opaque string | `User.AuthLinkRequested`의 전역 업무 멱등 키다. |
| AccountLifecycleStatus | enum | `provisioning`, `active`, `restricted`, `deactivated`, `provisioning_failed` |
| UserAuthStatus | enum | Auth 계약용 `active`, `restricted`, `deactivated` |
| RestrictionVersion | int64 | 1부터 시작하는 단조 증가 값이다. |

`AccountLifecycleStatus`는 사용자 서비스의 노출 가능 여부를, `UserAuthStatus`는 Auth가 Session 발급·차단에 사용하는 projection 값을 나타낸다. 두 값은 별도 의미를 가지지만 독립적으로 변경할 수 없다. `restricted -> restricted`, `deactivated -> deactivated`, 나머지 lifecycle 상태는 `active` auth 상태로 대응하며, `provisioning_failed`는 새 Auth 상태 Event를 만들지 않는다.

## Integration Event 계약

| 방향 | Event | version | Context 사용자 처리 |
| --- | --- | --- | --- |
| 수신 | `Auth.RegistrationVerificationCompleted` | 1 | 검증 binding과 프로필·동의 참조를 확인하고 Provisioning을 시작한다. |
| 발행 | `User.AuthLinkRequested` | 1 | 준비된 `user_id`와 Auth 상태를 binding에 묶어 전달한다. |
| 발행 | `User.AuthLinkRejected` | 1 | 프로필·동의 업무 규칙상 생성 거부일 때만 발행한다. |
| 수신 | `Auth.RegistrationUserLinked` | 1 | provisional 계정을 active로 전환한다. |
| 수신 | `Auth.RegistrationUserLinkRejected` | 1 | Provisioning을 실패로 닫고 보상 대상으로 표시한다. |
| 발행 | `User.AccountAuthStateChanged` | 1 | 계정 `auth_status`와 `restriction_version` 변경을 Auth에 전달한다. |
| 발행 | `User.ProfileUpdated` | 1 | 원문 없이 changed fields와 profile version만 전달한다. |
| 발행 | `User.ProfileMediaDetached` | 1 | 교체로 더는 참조하지 않는 이전 media asset의 정리를 미디어 시스템에 요청한다. |

Auth 가입 계약의 필드명과 의미는 [SD.A.30040](../../A_300_auth/A_300_40-api/README.md#가입-인증-완료와-사용자-계정-연동)을 그대로 따른다. 사용자 서비스가 별도 축약 이벤트를 만들지 않는다.

## User.AuthLinkRequested 필수 규칙

- `causation_id`는 소비한 `Auth.RegistrationVerificationCompleted.event_id`다.
- `auth_registration_id`, `registration_version`, binding ID/hash는 수신값을 바꾸지 않는다.
- `link_request_id`와 `user_id`는 전역 고유다.
- `user_auth_status`는 `active`, `restricted`, `deactivated` 중 하나다.
- `restriction_version`은 사용자 계정의 현재 단조 증가 version이다.
- 이메일·휴대폰·credential·challenge proof·프로필 원문을 넣지 않는다.

## User.AuthLinkRejected 필수 규칙

- 검증 이벤트를 업무 처리 대상으로 수락하는 순간 `link_request_id`를 발급하고 Provisioning에 저장한다. 사용자 계정이 아직 없어도 생략하지 않는다.
- 사용자 계정이 없는 거부 Event의 envelope `aggregate_id`는 `provisioning_id`를 사용한다. 존재하지 않는 `user_id`를 합성하지 않는다.
- `event_id`, 원인 Event의 `causation_id`, `link_request_id`, `auth_registration_id`, `registration_version`, binding ID/hash와 공개 가능한 `reason_code`만 전달한다.
- Provisioning 거부 상태와 거부 Outbox를 같은 트랜잭션에 저장한다. 같은 업무의 재시도는 같은 `link_request_id`와 기존 결과를 사용한다.
- producer·binding·payload가 충돌한 보안 거부와 transient 장애에는 이 업무 Event를 발행하지 않는다.

## Auth 결과 Event 검증

결과 Event에는 원래 verification binding/hash가 없고 `registration_version`은 Auth 처리 뒤 증가한 값이다. 따라서 가입 인증 완료 Event와 같은 필드의 정확 일치를 요구하지 않고 Event별 조건을 적용한다.

- `Auth.RegistrationUserLinked`는 검증된 Auth producer, `aggregate_id/auth_registration_id`, `correlation_id`, `link_request_id`, `user_id`, `causation_id=User.AuthLinkRequested.event_id`를 현재 Provisioning과 대조한다. 결과 `registration_version`은 저장한 binding snapshot version보다 크고 이전에 반영한 Auth 결과 version보다 낮지 않아야 한다.
- `Auth.RegistrationUserLinkRejected`의 `USER_CREATION_REJECTED`·`IDENTITY_OWNERSHIP_CONFLICT`는 `aggregate_id/auth_registration_id`, correlation, non-null `link_request_id`, 원인이 된 사용자 Event의 `causation_id`를 대조한다. 결과에 binding/hash가 없으므로 존재를 요구하지 않는다.
- `LINK_TIMEOUT`은 Auth가 연동 요청을 받지 못한 결과이므로 `link_request_id`와 `user_id`가 NULL일 수 있다. 이때는 검증된 Auth producer, `aggregate_id/auth_registration_id`, correlation, 저장한 `link_accept_until <= rejected_at`, 단조 증가한 결과 version으로 현재 nonterminal Provisioning을 찾는다.
- 같은 event ID는 Inbox 결과를 재생한다. 낮은 결과 version, terminal 상태를 되돌리는 결과, 다른 registration/correlation의 결과는 보안 감사와 함께 격리한다.

## User.AccountAuthStateChanged 계약

```json
{
  "eventId": "evt_01JUSERSTATE...",
  "eventType": "User.AccountAuthStateChanged",
  "eventVersion": 1,
  "occurredAt": "2026-07-10T10:00:00Z",
  "aggregateId": "00000000-0000-0000-0000-000000000001",
  "correlationId": "30d9fa85-0a18-4263-98b6-231dca5a6fb8",
  "causationId": "status_change_01JXYZ...",
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "status": "restricted",
    "restrictionVersion": 2,
    "reasonCode": "POLICY_VIOLATION",
    "effectiveAt": "2026-07-10T10:00:00Z"
  }
}
```

계약은 다음과 같이 확정한다.

- topic: `user.account-auth-state.v1`
- producer ACL: `user-service` workload만 produce
- Auth consumer group: `auth-user-auth-state-v1`
- 업무 키: `user_id`; 순서 기준: 단조 증가 `restriction_version`
- `causation_id`는 승인된 상태 변경 Command의 안정적인 `status_change_id`다. 누락된 causation은 Auth consumer가 계약 위반으로 거부한다.
- `restricted` 또는 `deactivated`를 받은 Auth는 더 높은 version일 때만 UserAuthState를 반영하고 active AccessGrant, 해당 사용자의 전체 Session 폐기, cache invalidation Outbox를 같은 트랜잭션에 저장한다.
- `active` 복구도 더 높은 version만 허용한다. 같거나 낮은 version은 기존 결과를 유지하며 다른 상태로 되돌리지 않는다.
- payload에는 운영자 ID, 승인 원문, 상세 사유를 넣지 않고 공개 가능한 `reason_code`만 전달한다.

이 계약은 [SD.A.30030의 UserAuthStateService](../../A_300_auth/A_300_30-service/README.md)와 [SD.A.30020의 UserAuthState 저장 규칙](../../A_300_auth/A_300_20-persistence/README.md)을 소비 기준으로 삼는다.

## User.ProfileMediaDetached 계약

- topic: `user.profile-media-detached.v1`
- producer ACL: `user-service` workload만 produce
- consumer: 프로필 미디어 시스템의 정리 worker
- payload: `user_id`, `detached_media_asset_id`, nullable `replacement_media_asset_id`, `profile_version`, `reason_code=PROFILE_IMAGE_REPLACED`, `detached_at`
- `causation_id`는 프로필 이미지 변경 Command ID이며, 프로필 원문과 signed/upload URL은 포함하지 않는다.
- 미디어 consumer는 event ID로 중복을 제거하고 asset의 owner/purpose와 현재 참조 상태를 재확인한 뒤 보존 정책에 따라 정리한다. 정리 실패는 Event 재시도 대상이며 완료되지 않은 삭제를 성공으로 기록하지 않는다.

## Principal과 개인정보 계약

- 외부가 보낸 사용자 헤더를 신뢰하지 않는다. Gateway/BFF가 외부 값을 제거하고 검증된 Principal을 다시 만든다.
- 목표 헤더는 `X-User-Id`, `X-User-Roles`, `X-Permission-Version`, `X-Token-Id`다.
- `X-User-Email`, 이메일/휴대폰 claim, 프로필 원문 header를 사용하지 않는다.
- 본인 변경은 Principal의 `user_id`와 리소스 소유자 `user_id`를 비교한다.
- 내부 운영 Command는 workload identity, canonical role, permission version, 최신 재인증·승인 참조를 함께 확인한다.

## 마이 조회 경계 결정

- `RM.A.01-05 마이 대시보드`는 여러 Context를 가로지르는 논리 Read Model이다.
- 물리 조합은 BFF가 수행한다. 사용자 서비스는 `UserMySlice`만 제공한다.
- BFF는 주문·배송·쿠폰·포인트·등급·관심·알림을 각각 조회하고 section별 `available`, `asOf`, `errorCode`를 표현한다.
- 사용자 서비스 DB와 event consumer는 외부 요약을 영구 복제하지 않는다.
- 사용자 서비스 장애는 마이의 사용자 section 실패로 표시하되, 가짜 기본 프로필로 성공 처리하지 않는다.

## Hotspot 처리표

| BC Hotspot | 이 설계의 처리 |
| --- | --- |
| 가입 입력 데이터 소유권 | 이름·닉네임 후보는 UserRegistrationDraft, 동의 receipt는 외부, 추천인 보상은 프로모션 Context로 분리한다. |
| 회원 등급 소유권 | 포인트·멤버십 Context 외부 값으로 유지한다. |
| 배송지 소유권 | 이번 사용자 서비스 원장에 포함하지 않고 후속 Context 결정으로 남긴다. |
| 마이 조합 위치와 부분 실패 | BFF 조합으로 결정하고 section별 상태 계약을 요구한다. |
| 탈퇴·제한·보존 정책 | 제한·비활성만 설계하고 탈퇴·삭제는 정책 확정 전 API를 만들지 않는다. |
| 닉네임·이미지 정책 | VO와 정책 Port를 두되 구체 운영값은 확인 필요로 남긴다. |
