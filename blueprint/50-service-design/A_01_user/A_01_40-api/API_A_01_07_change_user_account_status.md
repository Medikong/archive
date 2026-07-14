---
id: API.A.01-07
title: 사용자 계정 상태 변경 API
type: service-design-api-endpoint
status: draft
tags: [service-design, user, api, account-status]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
---

# API.A.01-07 사용자 계정 상태 변경

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/operator/users/{userId}/status-transitions` |
| 인증 | 운영자 웹 Session, CSRF, 허용 Origin, 최근 strong auth |
| 권한 | 현재 AccessGrant의 `user.account_status.change` |
| 멱등성 | `Idempotency-Key` 필수 |

## 요청

```json
{
  "targetStatus": "restricted",
  "reasonCode": "POLICY_VIOLATION",
  "expectedUserVersion": 3
}
```

## 성공 응답

```json
{
  "data": {
    "statusChangeId": "status_change_01JXYZ...",
    "userId": "00000000-0000-0000-0000-000000000001",
    "accountStatus": "restricted",
    "userVersion": 4,
    "changedAt": "2026-07-13T10:30:00Z",
    "userStatusChangeProof": "opaque-user-signed-proof"
  }
}
```

## 처리 규칙

- User 서비스는 운영 권한, 허용 전이와 expected version을 확인한다.
- 상태, 이력과 멱등 결과를 한 트랜잭션에 저장한다.
- User는 `userId`, `statusChangeId`, `accountStatus`, `userVersion`, `changedAt`, `audience=auth-service`와 만료 시각을 묶은 단기 `userStatusChangeProof`를 서명해 반환한다.
- 같은 멱등 요청을 다시 받으면 저장된 상태 변경 결과는 재사용하고 proof만 새 만료 시각으로 다시 발급할 수 있다.
- 운영 프론트엔드는 proof를 수정하지 않고 Auth API에 전달한다. 제한·비활성에서는 Auth가 Session을 폐기한다.
- 외부 Approval 서비스와 상태 변경 Event를 사용하지 않는다.

## 오류

- `403 USER_ACCOUNT_STATUS_PERMISSION_DENIED`
- `404 USER_NOT_FOUND`
- `409 USER_ACCOUNT_STATUS_TRANSITION_INVALID`
- `409 USER_VERSION_CONFLICT`
- `409 USER_IDEMPOTENCY_CONFLICT`

## 연관 시퀀스

- [SCN.A.01-04 사용자 계정 상태 변경](../A_01_50-sequence/SCN_A_01_04_change_user_status.md)
