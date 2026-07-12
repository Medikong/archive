---
id: API.A.01-08
title: 내부 사용자 계정 상태 조회 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
updated: 2026-07-10
---

# API.A.01-08 내부 사용자 계정 상태 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/internal/users/{userId}/account-status` |
| 역할 | Context 참여, 주문, 고객 지원 등 허가된 내부 서비스에 좁은 계정 상태를 제공한다. |
| 인증 | allowlist workload identity + 최소 permission |
| 캐시 | versioned Event projection 우선, 동기 조회는 보조 |

## 성공 응답

`200 OK`

```json
{
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "status": "active",
    "restrictionVersion": 1,
    "accountVersion": 3,
    "effectiveAt": "2026-07-10T08:00:02Z"
  }
}
```

## 처리 규칙

- 프로필, 이메일, 휴대폰, Session, 상세 운영 사유를 반환하지 않는다.
- 호출 서비스는 이 응답만으로 주문·판매자 소유권을 판단하지 않는다.
- 보안 게이트는 가능하면 `User.AccountAuthStateChanged` version projection을 사용한다.
- 사용자 없음과 접근 불가 오류가 개인정보 탐색 수단이 되지 않게 내부 권한을 먼저 검증한다.

## 오류

- `403 USER_INTERNAL_WORKLOAD_FORBIDDEN`
- `404 USER_NOT_FOUND`
- `503 USER_SERVICE_UNAVAILABLE`

## 도메인 매핑

`UserAccount`, `UserAccountStatusView`, `UserAccountQueryService`
