---
id: API.A.01-02
title: 본인 프로필 조회 API
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

# API.A.01-02 본인 프로필 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/users/me/profile` |
| 인증 | 사용자 Principal |

## 성공 응답

```json
{
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "accountStatus": "active",
    "nickname": "dropfan",
    "introduction": null,
    "profileMediaAssetId": null,
    "userVersion": 1,
    "updatedAt": "2026-07-13T10:00:01Z"
  }
}
```

signed URL은 포함하지 않는다. 프로필 이미지 컴포넌트가 Ingress를 통해 Media에 요청한다.

## 연관 시퀀스

- [SCN.A.01-05 마이 화면 컴포넌트 독립 조회](../A_01_50-sequence/SCN_A_01_05_load_my_components.md)

## 오류

- `401 USER_AUTHENTICATION_REQUIRED`
- `404 USER_NOT_FOUND`
