---
id: API.A.01-07
title: 사용자 계정 상태 변경 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
updated: 2026-07-10
---

# API.A.01-07 사용자 계정 상태 변경

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/internal/users/{userId}/status-transitions` |
| 역할 | 승인된 제한·해제·비활성 상태 전이를 실행한다. |
| 인증 | 운영 workload, canonical role/permission, 최신 재인증·승인 |
| 멱등성 | `Idempotency-Key` 필수 |

## 요청

```json
{
  "targetStatus": "restricted",
  "reasonCode": "POLICY_VIOLATION",
  "approvalRef": "approval_01JXYZ...",
  "expectedAccountVersion": 3
}
```

## 성공 응답

`200 OK`

```json
{
  "data": {
    "statusChangeId": "status_change_01JXYZ...",
    "userId": "00000000-0000-0000-0000-000000000001",
    "accountStatus": "restricted",
    "authStatus": "restricted",
    "restrictionVersion": 2,
    "accountVersion": 4,
    "effectiveAt": "2026-07-10T10:00:00Z"
  }
}
```

## 처리 규칙

- 허용 상태 전이와 approval scope를 검증한다.
- account row를 잠금 조회하고 expected version을 확인한다.
- `restrictionVersion`은 `authStatus`가 변경될 때 정확히 1 증가한다.
- actor scope의 멱등 실행마다 안정적인 `statusChangeId`를 한 번 발급한다.
- account, append-only status history, `causationId=statusChangeId`인 Auth 상태 Event outbox를 같은 transaction에 저장한다.
- `200`은 사용자 원장 변경과 전달할 Outbox의 영속화를 뜻하며 Auth의 Session 폐기 완료 증거는 아니다. 운영 화면은 restriction projection lag 경보를 별도로 확인한다.
- 상세 사유와 운영자 프로필을 Event에 넣지 않는다.

## 오류

- `403 USER_ACCOUNT_STATUS_PERMISSION_DENIED`
- `403 USER_ACCOUNT_STATUS_APPROVAL_INVALID`
- `409 USER_ACCOUNT_STATUS_TRANSITION_INVALID`
- `409 USER_ACCOUNT_VERSION_CONFLICT`
- `409 USER_IDEMPOTENCY_CONFLICT`
- `404 USER_NOT_FOUND`

## 도메인 매핑

`CMD.A.01-22`, `UserAccount`, `AccountStatusChange`, `EVT.A.01-27`
