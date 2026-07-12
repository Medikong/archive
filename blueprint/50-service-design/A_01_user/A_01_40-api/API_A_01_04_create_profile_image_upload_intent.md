---
id: API.A.01-04
title: 프로필 이미지 업로드 Intent 생성 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
updated: 2026-07-10
---

# API.A.01-04 프로필 이미지 업로드 Intent 생성

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/users/me/profile-image-upload-intents` |
| 역할 | 미디어 시스템의 프로필 전용 asset과 단기 upload URL을 준비한다. |
| 인증 | 사용자 Principal |
| 멱등성 | `Idempotency-Key` 필수 |

## 요청

```json
{
  "contentType": "image/jpeg",
  "contentLength": 245112
}
```

## 성공 응답

`201 Created`

```json
{
  "data": {
    "mediaAssetId": "asset_01JXYZ...",
    "uploadUrl": "https://media.example/upload/...",
    "expiresAt": "2026-07-10T10:10:00Z"
  }
}
```

## 처리 규칙

- 첫 로컬 트랜잭션에서 actor scope의 멱등 record와 안정적인 `mediaOperationId`를 선점한다.
- user ID, purpose=`user_profile`, 허용 content metadata와 `mediaOperationId`를 Media `CreateOrGet` Port의 멱등 키로 전달한다. 외부 호출 중 DB 트랜잭션을 열어 두지 않는다.
- Media 성공 뒤 같은 record에 media intent/asset 참조와 만료 시각만 저장하고 completed로 닫는다. 성공 뒤 프로세스가 죽어도 같은 operation ID로 기존 intent를 복구한다.
- binary는 사용자 서비스를 통과하지 않는다.
- upload URL은 DB에 저장하지 않는다.
- upload URL 응답은 `Cache-Control: private, no-store`를 사용한다.
- 이 응답만으로 프로필 이미지를 변경하지 않는다. API.A.01-05가 asset 검증 뒤 연결한다.
- 같은 actor/key 재시도는 media asset/intent 참조를 재사용하고 새 asset을 만들지 않는다. 미디어 Port에서 아직 유효한 URL을 다시 확인해 반환한다.
- 기존 intent가 만료됐으면 같은 key로 새 intent를 만들지 않고 `409 USER_PROFILE_UPLOAD_INTENT_EXPIRED`를 반환한다. 새 업로드는 새 key로 명시적으로 시작한다.

## 오류

- `401 USER_AUTHENTICATION_REQUIRED`
- `409 USER_ACCOUNT_NOT_ACTIVE`
- `409 USER_PROFILE_UPLOAD_INTENT_EXPIRED`
- `422 USER_PROFILE_MEDIA_TYPE_NOT_ALLOWED`
- `422 USER_PROFILE_MEDIA_TOO_LARGE`
- `503 USER_PROFILE_MEDIA_UNAVAILABLE`

## 도메인 매핑

`ProfileMediaPolicy`, `MediaAssetId`, `CreateProfileImageUploadIntentHandler`
