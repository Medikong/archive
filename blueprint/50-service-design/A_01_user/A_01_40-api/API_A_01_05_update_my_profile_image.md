---
id: API.A.01-05
title: 본인 프로필 이미지 연결 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
updated: 2026-07-10
---

# API.A.01-05 본인 프로필 이미지 연결

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `PUT /api/v1/users/me/profile-image` |
| 역할 | 검증 완료된 미디어 asset을 프로필에 연결한다. |
| 인증 | 사용자 Principal |
| 멱등성 | `Idempotency-Key` 필수 |
| 동시성 | 요청 본문의 `expectedProfileVersion` 필수 |

## 요청

```json
{
  "mediaAssetId": "asset_01JXYZ...",
  "expectedProfileVersion": 4
}
```

## 성공 응답

`200 OK`

```json
{
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "profileVersion": 5,
    "changedFields": ["profileImage"],
    "updatedAt": "2026-07-10T10:00:00Z"
  }
}
```

## 처리 규칙

- asset owner, purpose, scan/transform 완료를 미디어 시스템에서 확인한다.
- 검증 전 asset과 다른 사용자의 asset을 연결하지 않는다.
- actor scope의 IdempotencyRecord ID를 이미지 변경 Command와 정리 Event의 안정적인 causation ID로 사용한다.
- 변경 트랜잭션에서 UserAccount를 `FOR SHARE`, UserProfile을 `FOR UPDATE` 순서로 잠근 뒤 account active와 expected profile version을 다시 확인한다.
- profile update와 Event outbox를 같은 transaction에 저장한다.
- 이전 asset 정리는 `User.ProfileMediaDetached` version 1 Outbox로 `user.profile-media-detached.v1`에 전달한다. 비동기 정리 실패는 재시도하며 새 프로필 성공을 롤백하지 않는다.
- 멱등 재생은 비민감 Command 결과만 반환한다. 표시용 signed URL은 API.A.01-02를 다시 조회한다.

## 오류

- `403 USER_PROFILE_MEDIA_FORBIDDEN`
- `409 USER_PROFILE_MEDIA_NOT_READY`
- `409 USER_PROFILE_VERSION_CONFLICT`
- `409 USER_IDEMPOTENCY_CONFLICT`
- `503 USER_PROFILE_MEDIA_UNAVAILABLE`

## 도메인 매핑

`CMD.A.01-21`, `UserProfile`, `MediaAssetId`, `EVT.A.01-26`
