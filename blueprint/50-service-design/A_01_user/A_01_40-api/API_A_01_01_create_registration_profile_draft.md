---
id: API.A.01-01
title: 가입 프로필 초안 생성 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
updated: 2026-07-10
---

# API.A.01-01 가입 프로필 초안 생성

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/user-registration-drafts` |
| 역할 | Auth 회원가입 전에 Context 사용자 프로필 입력을 저장하고 `profileRequestId`를 발급한다. |
| 인증 | 허가된 BFF workload identity; 아직 사용자 Principal은 없음 |
| 멱등성 | `Idempotency-Key` 필수; BFF workload + registration process scope |

## 요청

```json
{
  "registrationProcessId": "regproc_01JXYZ...",
  "name": "홍길동",
  "nickname": null,
  "referralAttributionId": "refattr_01JXYZ..."
}
```

`registrationProcessId`는 BFF가 가입 시작 때 한 번 발급한 고엔트로피 opaque ID다. BFF는 서버 측 조정 레코드에서 이 ID, Auth Intent, 이후 발급받을 `profileRequestId`, `agreementReceiptId`를 한 작업으로 묶는다. 추천인 코드 원문은 받지 않고 프로모션 Context에서 검증한 attribution ID만 전달한다.

## 성공 응답

`201 Created`

```json
{
  "data": {
    "profileRequestId": "profile_req_01JXYZ...",
    "expiresAt": "2026-07-11T08:00:00Z"
  },
  "meta": { "requestId": "30d9fa85-0a18-4263-98b6-231dca5a6fb8" }
}
```

## 처리 규칙

- name은 필수이며 암호화해 저장한다.
- nickname은 선택 입력이고 구조적 정책만 적용한다.
- 동의 receipt는 동의 Context가 별도로 발급하며 이 API 요청에 넣지 않는다.
- 같은 BFF workload + `registrationProcessId` scope에서 같은 key와 같은 canonical payload는 기존 초안을 반환한다. 다른 가입 작업의 같은 key와 결과를 공유하지 않는다.
- BFF는 같은 registration process에 귀속된 Auth Intent·profile request·agreement receipt만 Auth 가입 시작 API에 전달한다.
- TTL은 설정값이며 Auth의 가입 완료 기한보다 충분히 길어야 한다.

## 오류

- `422 USER_REGISTRATION_PROFILE_INVALID`
- `409 USER_IDEMPOTENCY_CONFLICT`
- `403 USER_BFF_WORKLOAD_FORBIDDEN`
- `503 USER_PROFILE_POLICY_UNAVAILABLE`

## 도메인 매핑

`UserRegistrationDraft`, `PrivateName`, `Nickname`, `CreateRegistrationProfileDraftHandler`
