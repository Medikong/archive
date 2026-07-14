---
id: API.A.01-03
title: 본인 프로필 수정 API
type: service-design-api-endpoint
status: draft
tags: [service-design, user, api, profile]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
---

# API.A.01-03 본인 프로필 수정

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `PATCH /api/v1/users/me/profile` |
| 인증 | 사용자 Principal |
| 멱등성 | `Idempotency-Key` 필수 |

## 요청

```json
{
  "expectedUserVersion": 1,
  "nickname": "new-dropfan",
  "introduction": null
}
```

필드 누락은 유지, 명시적 `null`은 제거다. private name과 프로필 이미지는 변경하지 않는다.

## 성공 응답

```json
{
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "userVersion": 2,
    "changedFields": ["nickname", "introduction"],
    "updatedAt": "2026-07-13T10:10:00Z"
  }
}
```

## 처리 규칙

- 프로필 정책을 검증한 뒤 한 번의 조건부 UPDATE를 실행한다.
- 별도 잠금 조회, 프로필 version과 변경 Event를 사용하지 않는다.

## 오류

- `409 USER_ACCOUNT_NOT_ACTIVE`
- `409 USER_VERSION_CONFLICT`
- `409 USER_IDEMPOTENCY_CONFLICT`
- `422 USER_PROFILE_POLICY_VIOLATION`

## 연관 시퀀스

- [SCN.A.01-02 본인 프로필 수정](../A_01_50-sequence/SCN_A_01_02_update_own_profile.md)
