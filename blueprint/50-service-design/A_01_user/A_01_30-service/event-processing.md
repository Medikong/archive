---
id: SD.A.0130.EVENTS
title: Context 사용자 이벤트 처리 설계
type: service-design-service
status: draft
tags: [service-design, user, event, inbox, outbox, worker]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
---

# Context 사용자 이벤트 처리 설계

## Consumer

| Event | Handler | 멱등 기준 | 결과 |
| --- | --- | --- | --- |
| `Auth.RegistrationVerificationCompleted` v1 | HandleRegistrationVerificationCompleted | event ID + auth registration + binding | Provisioning 시작 또는 업무 거부 |
| UserAccountCreated | AdvanceUserProvisioning | provisioning + account flag | account flag + 고정 profile Command outbox |
| DefaultProfileCreated | AdvanceUserProvisioning | provisioning + profile flag | link 요청 가능 여부 평가 |
| `Auth.RegistrationUserLinked` v1 | HandleRegistrationUserLinked | event ID + link request | account active |
| `Auth.RegistrationUserLinkRejected` v1 | HandleRegistrationUserLinkRejected | event ID + link request/binding | Provisioning terminal, 존재하는 account만 provisioning_failed |

## Producer

| Event | 발생 원장 | 소비자 |
| --- | --- | --- |
| `User.AuthLinkRequested` v1 | UserProvisioning | Context 인증 |
| `User.AuthLinkRejected` v1 | UserProvisioning | Context 인증 |
| `User.AccountAuthStateChanged` v1 | UserAccount | Context 인증, 권한 게이트 |
| `User.ProfileUpdated` v1 | UserProfile | BFF cache/허가된 소비자 |
| `User.ProfileMediaDetached` v1 | UserProfile | 프로필 미디어 정리 worker |
| `User.AccountActivated` v1 | UserAccount | 알림·혜택 등 후속 후보 |

후속 소비자가 없으면 business Event를 미리 발행하지 않는다. Event payload는 소비자 계약이 확인된 필드만 포함한다.

`User.AccountAuthStateChanged`는 `user.account-auth-state.v1`에 발행한다. `user-service`만 발행하고 Auth의 `auth-user-auth-state-v1` consumer가 더 높은 `restriction_version`을 반영한다. 제한·비활성은 Auth의 UserAuthState 변경, active AccessGrant와 전체 Session 폐기, cache invalidation을 한 트랜잭션으로 유발한다.

envelope의 `causation_id`는 승인된 상태 변경의 `status_change_id`다. producer, event type/version, aggregate/user ID, correlation과 causation을 모두 검증할 수 있게 비워 두지 않는다.

## Event 검증

- topic ACL/workload identity로 producer를 확인한다.
- event type과 version allowlist를 적용한다.
- envelope의 event/aggregate/correlation/causation ID를 검증한다.
- 같은 event ID의 다른 payload hash는 보안 충돌로 격리한다.
- `Auth.RegistrationVerificationCompleted`의 binding/version/hash는 수신 payload 내부 계약과 기존 Provisioning이 정확히 일치해야 한다.
- 이메일·휴대폰·proof·credential·private profile 값이 있으면 schema 위반으로 거부한다.

Auth 결과는 초기 verification Event와 schema가 다르므로 다음 규칙을 사용한다.

| 결과 | 필수 상관관계 | version 규칙 |
| --- | --- | --- |
| RegistrationUserLinked | registration, correlation, link request, user, causation=발행한 link-request Event | binding snapshot보다 크고 이전 결과보다 낮지 않음 |
| UserLinkRejected: 업무 거부/충돌 | registration, correlation, non-null link request, causation=발행한 사용자 Event | binding snapshot보다 크고 이전 결과보다 낮지 않음 |
| UserLinkRejected: timeout | registration, correlation, `rejected_at >= link_accept_until`; link/user NULL 허용 | binding snapshot보다 크고 이전 결과보다 낮지 않음 |

결과 Event에는 verification binding/hash가 없으므로 요구하지 않는다. 낮은 version이나 terminal 상태를 되돌리는 결과는 격리한다.

## Retry 분류

| 분류 | 예시 | 처리 |
| --- | --- | --- |
| transient | DB timeout, broker unavailable, agreement service timeout | backoff + deferred |
| concurrency | Aggregate version conflict | 현재 상태 재조회 후 제한 재시도 |
| business rejection | draft expired, required agreement invalid | `User.AuthLinkRejected` |
| security conflict | producer/binding/payload mismatch | rejected inbox + 보안 감사, 업무 상태 유지 |
| permanent delivery | outbox max attempts | dead + 온콜/수동 재처리 |

기술 실패를 업무 거부나 성공으로 변환하지 않는다.

## Reconciliation Worker

- `requested` Provisioning이 Auth 결과 없이 오래된 경우 Auth 계약의 기한·consumer lag를 확인한다.
- 같은 `link_request_id`를 재발행하고 새 ID를 만들지 않는다.
- 재발행은 원래 Outbox row를 다시 queue해 같은 `link_event_id/event_id`를 사용한다. 새 Event를 만들지 않는다.
- `not_requested` 상태에서 account가 없으면 저장된 Account Command ID를, account만 준비됐으면 저장된 Profile Command ID를 사용해 누락 Command를 재요청한다.
- `agreement_validation_status=pending`이 오래된 Provisioning은 Inbox lease/deferred 상태와 Auth 기한을 확인한다. terminal timeout 뒤에는 Agreement 결과나 생성 Command를 재개하지 않는다.
- `provisioning_failed` 정리 시 보존 정책과 crypto-shred 준비 여부를 확인한다.
- restriction outbox backlog가 임계치를 넘으면 Auth projection 지연 경보를 발생시킨다.
- local grace는 Auth의 `link_accept_until`을 연장하지 않는다. timeout 결과의 최종 판단은 Auth가 소유한다.

## 관측과 SLO 후보

| 지표 | 목적 |
| --- | --- |
| event consumer lag/oldest age | Auth 연동 지연 |
| provisioning state/age | 가입 장기 정체 |
| inbox rejected/dead total | 계약·보안 문제 |
| outbox pending/dead total | 전달 신뢰성 |
| auth link result/reason | 연동 성공·업무 거부 분석 |
| restriction projection lag | 계정 제한 보안 지연 |

trace에는 correlation ID만 제한적으로 사용하고 user/event ID를 metric label로 쓰지 않는다.

## Contract Test

- Auth의 JSON fixture와 사용자 consumer schema가 version 1에서 일치한다.
- `User.AuthLinkRequested` payload가 Auth contract의 required/forbidden 필드를 만족한다.
- 중복 event, 다른 payload, 뒤바뀐 순서, 기한 초과, transient 장애를 각각 검증한다.
- 성공 결과의 증가한 registration version, binding 없는 업무 거부, link/user가 NULL인 timeout fixture를 각각 검증한다.
- 계정 상태 version은 중복·낮은 version을 무시하고 더 높은 version만 전달한다.
- private name, 이메일, 휴대폰, profile text가 event payload에 없는지 검사한다.
- `User.ProfileMediaDetached`가 이전 asset opaque ID와 정리 metadata만 포함하고 미디어 consumer가 같은 event ID를 멱등 처리하는지 검증한다.
