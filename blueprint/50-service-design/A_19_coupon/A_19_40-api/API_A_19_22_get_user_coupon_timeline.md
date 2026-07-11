---
id: API.A.19-22
title: 사용자 쿠폰 타임라인 조회 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, cs, timeline, query]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-22 사용자 쿠폰 타임라인 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/internal/users/{userId}/coupon-timeline` |
| operationId | `getUserCouponTimeline` |
| 역할 | 승인된 CS 업무 범위에서 발급·사용·회수·만료 이력을 조회한다. |
| API 유형 | Query |
| 인증·권한 | CS·운영 workload, caseRef, 사용자 조회 권한 |
| 노출 범위 | internal |
| 멱등성 | 해당 없음 |
| 캐시 | `no-store` |
| 호환성 | cursor pagination |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_22_get_user_coupon_timeline.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-017`, `UC.A.19-18` |
| Read Model | `RM.A.19-06` |
| 관련 UC | [UC.A.04 CS 주문·쿠폰 지원](../../../30-uc/UC_A_04_cs_order_coupon_support.md) |

## 책임과 경계

- 발급 요청, 실제 발급, 예약·확정·해제·회수·만료 사건을 시간순으로 제공한다.
- CS 문의 원문, 주문·결제 상세와 사용자 프로필을 조합하지 않는다.
- 외부 원본은 caseRef와 결과 ExternalRef로만 연결한다.

## 보안과 개인정보

- caseRef와 사용자 조회 권한을 매 요청 검증하고 audit한다.
- 필요 최소 이벤트와 공개 사유만 반환하며 코드·payload 원문을 숨긴다.

## 처리 규칙

1. workload, caseRef와 target user 접근 범위를 검증한다.
2. 시간 범위, event type filter와 cursor를 적용한다.
3. 사건, 결과 ref와 `asOf`를 시간순으로 반환한다.

## 상태 변경과 트랜잭션

- 상태를 변경하지 않는다.
- 조회 감사는 쿠폰 Aggregate가 아니라 외부 감사 시스템에 남긴다.

## 멱등성과 동시성

- Query이므로 멱등 키가 필요 없다.
- `(user_id, occurred_at, event_id)` cursor로 안정적인 정렬을 보장한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| caseRef·권한 불충족 | 사용자 존재를 공개하지 않음 | 승인된 CS 작업을 준비한다. |
| projection 지연 | 사건 없음으로 위조하지 않음 | `asOf` 확인 뒤 재조회한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Service | 사용자 쿠폰 타임라인 Query |
| Read Model | `UserCouponTimeline` |
| 원천 Event | 발급·사용·만료·복구 Event |

## 관측성과 운영

- workload, caseRef hash, 조회 범위, 결과 수와 지연을 audit한다.
- user ID는 metric label에 넣지 않는다.

## 검증 항목

- caseRef 없는 조회와 범위 밖 사용자가 거절된다.
- 사건 순서와 result ref가 append-only 원장과 일치한다.
- 응답·audit에 불필요한 개인정보가 없다.

## 연관 시퀀스

- CS 지원 시퀀스에서 조회 뒤 release·revoke·보상 승인으로 이어질 수 있다.

## 호환성과 변경 정책

- 새 timeline event kind는 기존 클라이언트가 unknown을 표시할 수 있을 때 추가한다.

## 확인 필요

- `HOTSPOT.A.19-05`: CS 조회·위험 작업 승인 범위.
