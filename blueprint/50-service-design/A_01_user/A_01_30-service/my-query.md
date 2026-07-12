---
id: SD.A.0130.MY-QUERY
title: Context 사용자 마이 조회 경계
type: service-design-service
status: draft
tags: [service-design, user, my, bff, query]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
---

# Context 사용자 마이 조회 경계

## 결정

`RM.A.01-05 마이 대시보드`의 물리 조합은 BFF가 담당한다. 사용자 서비스는 `UserMySlice` Query만 제공한다.

## 이유

- 주문·쿠폰·포인트·등급·찜·알림은 각 서비스가 변경 권한과 최신성 기준을 가진다.
- 사용자 서비스가 모든 서비스를 호출하면 사용자 계정 책임과 화면 조합 책임이 결합되고 장애가 집중된다.
- PAGE.A.10은 쿠폰/포인트 조회 실패를 부분 상태로 표시하도록 요구하므로 section별 실패 계약이 필요하다.
- BFF는 화면별 timeout·cache·메뉴 정책을 조정할 수 있고 사용자 원장은 외부 데이터에 의존하지 않는다.

## 사용자 서비스 Query

`GetUserMySlice`는 다음 정보만 제공한다.

| Section | 필드 |
| --- | --- |
| account | userId, accountStatus, restrictionVersion, accountVersion |
| profile | nickname, displayName, introduction, profileImage availability/url/expiresAt, profileVersion |

private name은 마이 기본 slice에서 제외한다. 본인 프로필 편집 화면이 별도 Query로 요청한다.

## BFF 조합 계약

| Section | 원천 | 장애 시 |
| --- | --- | --- |
| user | user-service | user section unavailable, 로그인 재검증 또는 재시도 |
| orderSummary/recentOrder | order·shipping | 해당 section unavailable, 주문 수를 0으로 위조하지 않음 |
| coupon | coupon-service | available=false + stable error code |
| point/membership | point·membership | 배지·진행 정보 숨김, 기준 시각 표시 |
| interest | interest-service | 찜 메뉴는 유지하되 count 미표시 가능 |
| notification | notification-service | unread 상태 unknown |
| invitation | promotion-service | 배너 숨김 또는 unavailable |
| menus | BFF 화면 정책 | 활성 여부와 이동 경로 제공 |

Section 공통 형식 후보:

```json
{
  "available": false,
  "asOf": "2026-07-10T10:00:00Z",
  "errorCode": "MY_COUPON_TEMPORARILY_UNAVAILABLE",
  "data": null
}
```

## Timeout과 Cache

- BFF가 전체 deadline을 정하고 section별 더 짧은 timeout을 배분한다.
- 사용자 서비스는 order/coupon 등 downstream timeout을 알지 않는다.
- BFF cache는 source, `asOf`, account/profile version을 포함하며 signed image URL의 만료 시각을 넘겨 재사용하지 않는다.
- account restricted/deactivated Event는 user section cache를 무효화하지만 Auth 차단은 별도 versioned projection으로 즉시 적용한다.
- 실패 section을 장기 stale success로 재생하지 않는다.

## 보안

- BFF는 검증된 Principal의 user ID로 사용자 slice를 조회한다.
- 클라이언트가 전달한 user ID path/query로 본인 범위를 바꾸지 않는다.
- 내부 endpoint는 BFF workload identity와 user principal을 모두 요구한다.
- 각 외부 section은 같은 user ID라도 자체 리소스 소유권을 다시 확인한다.

## 확인 필요

- BFF 전체·section별 timeout과 cache TTL
- section 오류 코드와 클라이언트 표시 문구
- 메뉴·프로모션 화면 정책 저장 위치
- user-service 장애 시 마이 탭 로그인 화면 처리
