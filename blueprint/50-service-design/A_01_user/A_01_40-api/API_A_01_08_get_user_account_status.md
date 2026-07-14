---
id: API.A.01-08
title: 사용자 계정 상태 조회 API
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

# API.A.01-08 사용자 계정 상태 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/operator/users/{userId}/status` |
| 인증 | 운영자 웹 Session과 최근 strong auth |
| 권한 | 현재 AccessGrant의 `user.account_status.read` |

## 성공 응답

```json
{
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "accountStatus": "restricted",
    "userVersion": 4,
    "updatedAt": "2026-07-13T10:30:00Z"
  }
}
```

## 처리 규칙

- User의 현재 상태만 반환한다.
- Auth의 Session·IdentityLink와 운영 상세 사유는 반환하지 않는다.
- Auth는 User가 서명한 상태 변경 proof를 자체 원장에 멱등 적용한다.

## 오류

- `403 USER_FORBIDDEN`
- `404 USER_NOT_FOUND`
