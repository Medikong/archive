---
id: API.A.19-03
title: 내 쿠폰 목록 조회 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, wallet, query]
source: local
created: 2026-07-11
updated: 2026-07-12
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-03 내 쿠폰 목록 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/users/me/coupons` |
| operationId | `listMyCoupons` |
| 역할 | 본인의 쿠폰함을 상태별로 조회한다. |
| API 유형 | Query |
| 인증·권한 | 사용자 Principal의 본인 범위 |
| 노출 범위 | public |
| 멱등성 | 해당 없음 |
| 캐시 | `private, no-store` |
| 호환성 | cursor pagination |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_03_list_my_coupons.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-006`, `FR-012~013`, `UC.A.19-01~02` |
| Read Model | `RM.A.19-01`, 활성 `RM.A.19-09` |
| 영속성 | [조회 모델과 인덱스](../A_19_20-persistence/read-models-and-indexes.md) |

## 책임과 경계

- 발급 요청, 사용자 쿠폰과 사용 Event를 합성한 쿠폰함을 반환한다.
- `pending`, `available`, `reserved`, `used`, `reclaimed`, `expired`, `failed` 필터를 제공한다.
- 주문·상품 원본 상태를 실시간으로 대신하지 않고 투영 Snapshot과 기준 시각을 반환한다.

## 보안과 개인정보

- URL이나 body에서 다른 `user_id`를 받지 않는다.
- 캠페인 내부 승인·실패 사유, 외부 원본 참조와 코드 정보를 숨긴다.
- 활성 읽기 전용 안내는 현재 사용자의 적용 범위에 해당하는 문구만 반환한다.

## 처리 규칙

1. Principal의 `user_id`와 status·cursor·limit를 검증한다.
2. `(user_id, wallet_status, sort_at, projection_id)` cursor로 Read Model을 조회한다.
3. 최대 page size를 적용하고 `nextCursor`, `asOf`, 활성 안내를 반환한다.

## 상태 변경과 트랜잭션

- 상태를 변경하지 않는다.
- 투영 지연을 성공 값으로 감추지 않고 `asOf`를 제공한다.

## 멱등성과 동시성

- Query이므로 Idempotency-Key를 받지 않는다.
- cursor는 정렬 키와 projection ID를 포함해 같은 시각 항목의 누락·중복을 막는다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| cursor 위변조·만료 | 내부 정렬 키를 공개하지 않음 | 첫 페이지부터 다시 조회한다. |
| 투영 저장소 장애 | 빈 목록으로 위조하지 않음 | 같은 Query를 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Service | 쿠폰함 Query |
| Read Model | `BuyerCouponWallet`, `CouponReadOnlyNotice` |
| Repository | Read Model별 Query Repository |
| Event 원천 | 발급·사용·만료·운영 제어 Event |

## 관측성과 운영

- route, status filter, page size, 결과 수와 projection lag를 기록한다.
- user ID, cursor와 쿠폰 ID는 metric label에 넣지 않는다.

## 검증 항목

- 다른 사용자의 쿠폰이 섞이지 않는다.
- 경계 cursor에서 누락·중복이 없다.
- 발급 대기와 실제 보유, 예약과 사용 완료가 원장 사건에 맞게 투영된다.

## 연관 시퀀스

- [PAGE.A.19](../../../10-sitemap/buyer-mobile-web/PAGE_A_19_coupon_wallet/PAGE_A_19_owned_coupon.md)의 목록 조회에 연결된다.

## 호환성과 변경 정책

- 새 wallet status 추가는 클라이언트의 unknown 처리 정책과 함께 배포한다.

## 결정 반영

- `HOTSPOT.A.19-01`: `pending`은 `발급 대기`, 실제 발급이 끝난 `completed`는 `발급 완료`로 표시한다. 접수·완료·실패를 같은 상태로 합치지 않는다.
