---
id: API.A.01-06
title: 마이 페이지 사용자 조각 조회 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
updated: 2026-07-10
---

# API.A.01-06 마이 페이지 사용자 조각 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/internal/users/{userId}/my-slice` |
| 역할 | BFF가 마이를 조합할 때 사용자 계정·프로필 section만 제공한다. |
| 인증 | BFF workload identity + 사용자 Principal |
| 캐시 | signed URL 포함 응답은 `Cache-Control: private, no-store`; BFF 캐시는 URL 만료 전까지만 별도 정책 적용 |

## 성공 응답

`200 OK`

```json
{
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "accountStatus": "active",
    "restrictionVersion": 1,
    "accountVersion": 3,
    "profile": {
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
    }
  }
}
```

## 처리 규칙

- workload가 BFF allowlist에 있고 Principal user ID와 path userId가 같아야 한다.
- private name은 반환하지 않는다.
- 주문·배송·쿠폰·포인트·등급·찜·알림·메뉴·배너를 반환하지 않는다.
- 외부 section 실패 처리는 BFF가 담당한다.
- BFF가 사용자 조각을 캐시하면 account/profile version과 `asOf`를 기록하고 signed URL 만료 시각을 넘겨 재사용하지 않는다.

## 오류

- `401 USER_AUTHENTICATION_REQUIRED`
- `403 USER_BFF_WORKLOAD_FORBIDDEN`
- `403 USER_RESOURCE_OWNER_MISMATCH`
- `404 USER_NOT_FOUND`
- `409 USER_ACCOUNT_NOT_ACTIVE`

## 도메인 매핑

서비스 설계에서 분리한 `GetUserMySlice` Query, `UserMySlice`, `GetUserMySliceHandler`. BC의 `CMD.A.01-23 마이 대시보드 조회`는 BFF의 전체 조합 책임으로 유지한다.
