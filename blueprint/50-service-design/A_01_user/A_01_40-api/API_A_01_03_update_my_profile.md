---
id: API.A.01-03
title: 본인 프로필 수정 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
updated: 2026-07-10
---

# API.A.01-03 본인 프로필 수정

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `PATCH /api/v1/users/me/profile` |
| 역할 | 닉네임과 인사말을 부분 수정한다. |
| 인증 | 사용자 Principal |
| 멱등성 | `Idempotency-Key` 필수 |
| 동시성 | 요청 본문의 `expectedProfileVersion` 필수 |

## 요청

```json
{
  "expectedProfileVersion": 4,
  "nickname": "new-dropfan",
  "introduction": null
}
```

필드 누락은 유지, 명시적 NULL은 제거다. private name과 프로필 이미지는 이 API에서 변경하지 않는다.

## 성공 응답

`200 OK`

```json
{
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "profileVersion": 5,
    "changedFields": ["nickname", "introduction"],
    "updatedAt": "2026-07-10T10:00:00Z"
  }
}
```

멱등 재생은 이 비민감 Command 결과를 반환한다. 전체 프로필과 새 signed URL이 필요하면 API.A.01-02를 다시 조회한다.

## 처리 규칙

- Principal과 profile owner가 같아야 한다.
- UserAccount를 `FOR SHARE`, UserProfile을 `FOR UPDATE` 순서로 잠근 뒤 account active와 expected profile version을 다시 확인한다. 동시 제한 트랜잭션이 먼저 커밋되면 변경을 거부한다.
- nickname/introduction 정책 version을 적용한다.
- 요청 본문의 expected profile version이 일치해야 한다.
- 변경과 idempotency record, `User.ProfileUpdated` outbox를 같은 transaction에 저장한다.

## 오류

- `401 USER_AUTHENTICATION_REQUIRED`
- `409 USER_ACCOUNT_NOT_ACTIVE`
- `409 USER_PROFILE_VERSION_CONFLICT`
- `409 USER_IDEMPOTENCY_CONFLICT`
- `422 USER_PROFILE_POLICY_VIOLATION`

## 도메인 매핑

`CMD.A.01-20`, `UserProfile`, `Nickname`, `Introduction`, `EVT.A.01-25`
