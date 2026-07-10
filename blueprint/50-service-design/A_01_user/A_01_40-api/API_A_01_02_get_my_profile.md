---
id: API.A.01-02
title: 본인 프로필 조회 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
updated: 2026-07-10
---

# API.A.01-02 본인 프로필 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/users/me/profile` |
| 역할 | 로그인 사용자의 private profile과 표시 프로필을 조회한다. |
| 인증 | 사용자 Principal |
| 캐시 | `Cache-Control: private, no-store`; 조건부 `304` 미지원 |

## 성공 응답

`200 OK`

```json
{
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "name": "홍길동",
    "nickname": "dropfan",
    "displayName": "dropfan",
    "introduction": "한정 드롭을 기다리는 중",
    "profileImage": {
      "available": true,
      "url": "https://media.example/signed/...",
      "expiresAt": "2026-07-10T10:10:00Z",
      "errorCode": null
    },
    "profileVersion": 4
  },
  "meta": { "requestId": "30d9fa85-0a18-4263-98b6-231dca5a6fb8" }
}
```

## 처리 규칙

- path/query의 user ID를 받지 않고 Principal user ID만 사용한다.
- active 계정만 일반 응답한다.
- signed image URL은 조회 시 발급하고 저장하지 않는다.
- private name과 회전하는 signed URL을 포함하므로 profile version을 HTTP representation ETag로 사용하지 않는다. 변경 동시성은 요청 본문의 `expectedProfileVersion`으로 처리한다.
- 이미지 URL 발급 실패는 `200`의 `profileImage.available=false`, nullable URL/만료 시각, 안정적인 `errorCode`로 표시한다. private profile을 임의 기본값으로 바꾸지 않는다.

## 오류

- `401 USER_AUTHENTICATION_REQUIRED`
- `404 USER_PROFILE_NOT_FOUND`
- `409 USER_ACCOUNT_NOT_ACTIVE`

## 도메인 매핑

`UserAccount`, `UserProfile`, `UserProfileView`, `GetOwnProfileHandler`
