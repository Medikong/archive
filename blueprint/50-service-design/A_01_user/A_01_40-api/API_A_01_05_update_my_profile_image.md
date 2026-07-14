---
id: API.A.01-05
title: 본인 프로필 이미지 연결 API
type: service-design-api-endpoint
status: draft
tags: [service-design, user, api, profile-image]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
---

# API.A.01-05 본인 프로필 이미지 연결

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `PUT /api/v1/users/me/profile-image` |
| 인증 | 사용자 Principal |
| 멱등성 | `Idempotency-Key` 필수 |

## 요청

```json
{
  "mediaAssetId": "asset_01JXYZ...",
  "mediaAssetProof": "opaque-signed-proof",
  "expectedUserVersion": 2
}
```

## 성공 응답

```json
{
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "profileMediaAssetId": "asset_01JXYZ...",
    "userVersion": 3,
    "updatedAt": "2026-07-13T10:20:00Z"
  }
}
```

## 처리 규칙

- Media proof의 asset ID, owner user ID, purpose, 검사 완료와 만료를 검증한다.
- 한 번의 조건부 UPDATE로 자산 ID와 `user_version`을 변경한다.
- upload intent와 이전 자산 정리는 Media 책임이다.

## 오류

- `403 USER_PROFILE_MEDIA_PROOF_INVALID`
- `409 USER_ACCOUNT_NOT_ACTIVE`
- `409 USER_VERSION_CONFLICT`
- `409 USER_IDEMPOTENCY_CONFLICT`

## 연관 시퀀스

- [SCN.A.01-03 프로필 이미지 업로드와 연결](../A_01_50-sequence/SCN_A_01_03_profile_image_upload_attach.md)
