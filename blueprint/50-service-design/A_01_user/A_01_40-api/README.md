---
id: SD.A.0140
title: Context 사용자 API 설계
type: service-design-api
status: draft
tags: [service-design, user, api, rest, event-contract]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
bounded_context: BC.A.01
domain_model: SD.A.0110
persistence: SD.A.0120
service: SD.A.0130
---

# Context 사용자 API 설계

## 역할

가입 프로필 초안, 본인 프로필, BFF 사용자 조각, 내부 계정 상태 API와 Auth 연동 Event 계약을 정의한다.

## 공통 계약

- 기본 응답은 JSON, 오류는 `application/problem+json`을 사용한다.
- 보호 API는 Gateway/BFF가 만든 검증된 Principal만 신뢰한다.
- 목표 사용자 헤더는 `X-User-Id`, `X-User-Roles`, `X-Permission-Version`, `X-Token-Id`다.
- `X-User-Email`, 이메일/휴대폰 claim, 외부가 직접 보낸 `X-User-*` header를 사용하지 않는다.
- 생성·변경 API는 `Idempotency-Key`, 모든 요청은 `X-Request-Id`와 W3C `traceparent`를 지원한다.
- 멱등 범위는 `API ID + actor 또는 registration process + key`다. 본인 API는 `user_id`, 가입 초안은 BFF workload + `registrationProcessId`, 운영 API는 operator/workload + target user ID에 묶는다.
- 저장 결과를 재생하기 전 현재 Principal/workload, 소유권, 계정 상태와 승인을 다시 확인한다.
- 프로필 변경은 요청 본문의 `expectedProfileVersion`을 요구하고 불일치는 `409 USER_PROFILE_VERSION_CONFLICT`로 반환한다.
- 내부 API는 workload identity와 최소 권한을 추가로 검증한다.
- 웹 unsafe method는 Gateway/BFF가 허용 `Origin`과 현재 Session에 묶인 `X-CSRF-Token`을 검증한 뒤에만 Principal을 생성한다. 검증 증거가 없거나 만료되면 사용자 서비스 호출 전에 fail closed한다.
- private name 또는 단기 signed URL을 포함한 응답은 `Cache-Control: private, no-store`를 사용한다. signed URL에는 `expiresAt`을 함께 반환하며 조건부 `304` 응답에 재사용하지 않는다.

오류 예시:

```json
{
  "type": "https://docs.dropmong.dev/problems/user-profile-version-conflict",
  "title": "User profile version conflict",
  "status": 409,
  "code": "USER_PROFILE_VERSION_CONFLICT",
  "requestId": "30d9fa85-0a18-4263-98b6-231dca5a6fb8"
}
```

민감 입력, 내부 user ID 충돌 상대, private name, approval 원문은 problem detail에 넣지 않는다.

## REST API

| API | Method / Path | 책임 | 인증 |
| --- | --- | --- | --- |
| [API.A.01-01](API_A_01_01_create_registration_profile_draft.md) | `POST /api/v1/user-registration-drafts` | 가입 전 프로필 초안 생성 | BFF workload identity |
| [API.A.01-02](API_A_01_02_get_my_profile.md) | `GET /api/v1/users/me/profile` | 본인 프로필 조회 | 사용자 Principal |
| [API.A.01-03](API_A_01_03_update_my_profile.md) | `PATCH /api/v1/users/me/profile` | 닉네임·인사말 수정 | 사용자 Principal |
| [API.A.01-04](API_A_01_04_create_profile_image_upload_intent.md) | `POST /api/v1/users/me/profile-image-upload-intents` | 프로필 이미지 업로드 준비 | 사용자 Principal |
| [API.A.01-05](API_A_01_05_update_my_profile_image.md) | `PUT /api/v1/users/me/profile-image` | 검증된 미디어 자산 연결 | 사용자 Principal |
| [API.A.01-06](API_A_01_06_get_my_page_user_slice.md) | `GET /api/v1/internal/users/{userId}/my-slice` | BFF용 사용자 조각 | BFF workload + 사용자 Principal |
| [API.A.01-07](API_A_01_07_change_user_account_status.md) | `POST /api/v1/internal/users/{userId}/status-transitions` | 제한·해제·비활성 | 운영 workload + 권한·승인 |
| [API.A.01-08](API_A_01_08_get_user_account_status.md) | `GET /api/v1/internal/users/{userId}/account-status` | 좁은 내부 계정 상태 조회 | 허가된 workload |

사용자 생성 REST API와 lazy `ensure` API는 제공하지 않는다. 사용자 생성은 Auth integration event만 시작할 수 있다.

## 마이 페이지 경계

- 사용자 서비스에는 `/dashboard` 전체 조합 API를 만들지 않는다.
- `API.A.01-06`은 사용자 계정·프로필 조각만 반환한다.
- 주문·배송·쿠폰·포인트·등급·찜·알림·프로모션 section은 BFF가 각 원천 API에서 조합한다.
- BFF는 section별 `available`, `asOf`, `errorCode`를 제공하고 실패 값을 0으로 위조하지 않는다.

## Auth Integration Event

### 수신: Auth.RegistrationVerificationCompleted version 1

필수 필드:

- envelope: eventId, eventType, eventVersion, occurredAt, aggregateId, correlationId, causationId
- data: registrationVersion, verificationBindingId, verificationSnapshotHash, profileRequestId, agreementReceiptId, assuranceSummary, linkAcceptUntil

### 발행: User.AuthLinkRequested version 1

```json
{
  "eventId": "evt_01JUSER...",
  "eventType": "User.AuthLinkRequested",
  "eventVersion": 1,
  "occurredAt": "2026-07-10T08:00:01Z",
  "aggregateId": "00000000-0000-0000-0000-000000000001",
  "correlationId": "30d9fa85-0a18-4263-98b6-231dca5a6fb8",
  "causationId": "evt_01JAUTH...",
  "data": {
    "linkRequestId": "ulnk_01JXYZ...",
    "authRegistrationId": "reg_01JXYZ...",
    "verificationBindingId": "vbind_01JXYZ...",
    "verificationSnapshotHash": "sha256-base64url-value",
    "registrationVersion": 4,
    "userId": "00000000-0000-0000-0000-000000000001",
    "userAuthStatus": "active",
    "restrictionVersion": 1,
    "issuedAt": "2026-07-10T08:00:01Z"
  }
}
```

이메일·휴대폰·profile 원문·challenge proof·credential은 금지한다. `User.AuthLinkRejected`는 업무 규칙상 사용자 생성 거부에만 사용하고 transient 장애에는 발행하지 않는다.

### 수신 결과

- `Auth.RegistrationUserLinked` version 1: `authRegistrationId`, Auth 처리 뒤 증가한 `registrationVersion`, `linkRequestId`, `userId`, `linkedAt`. `causationId`는 발행한 `User.AuthLinkRequested.eventId`와 대조하며 결과 version을 원래 snapshot version과 같다고 요구하지 않는다.
- `Auth.RegistrationUserLinkRejected` version 1: `authRegistrationId`, 증가한 `registrationVersion`, nullable `linkRequestId/userId`, 공개 `reasonCode`, `rejectedAt`. 결과에는 verification binding/hash가 없다.
- `USER_CREATION_REJECTED`·`IDENTITY_OWNERSHIP_CONFLICT`는 non-null link request와 원인이 된 사용자 Event로 상관시킨다. `LINK_TIMEOUT`은 link/user가 NULL일 수 있으므로 registration/correlation, `rejectedAt >= linkAcceptUntil`, 단조 증가한 version으로 현재 작업을 찾는다.

## 공통 오류

| Code | HTTP | 의미 |
| --- | --- | --- |
| USER_AUTHENTICATION_REQUIRED | 401 | 검증된 Principal 없음 |
| USER_FORBIDDEN | 403 | 본인·workload·권한 조건 불충족 |
| USER_NOT_FOUND | 404 | 노출 가능한 범위에서 사용자 없음 |
| USER_ACCOUNT_NOT_ACTIVE | 409 | provisioning/restricted/deactivated |
| USER_IDEMPOTENCY_CONFLICT | 409 | 같은 key의 다른 payload |
| USER_VERSION_CONFLICT | 409 | 계정·프로필 version 충돌 |
| USER_INPUT_INVALID | 422 | 도메인 입력 정책 위반 |
| USER_DEPENDENCY_UNAVAILABLE | 503 | 미디어·동의 등 필수 의존성 장애 |

## 관측성

- endpoint별 request count/latency/status, dependency latency, event lag를 기록한다.
- route template, method, stable error code만 metric label로 사용한다.
- user ID, request ID, event ID, profile value는 metric label에 넣지 않는다.
- 로그에는 request/correlation ID와 내부 결과 코드만 남기고 민감 값을 마스킹한다.
