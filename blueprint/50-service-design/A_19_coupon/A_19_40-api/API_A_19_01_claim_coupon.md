---
id: API.A.19-01
title: 쿠폰 수령 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, claim, issuance]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-01 쿠폰 수령

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/coupon-campaigns/{campaignId}/claims` |
| operationId | `claimCoupon` |
| 역할 | 구매자의 직접 수령 요청을 발급 요청으로 접수한다. |
| API 유형 | Command |
| 인증·권한 | 사용자 Principal의 `user_id`; 본인 요청만 허용 |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `private, no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_01_claim_coupon.yaml)

요청·응답 필드, `202` 상태와 공개 오류 wire 계약은 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-003~005`, `UC.A.19-04` |
| BC | `CMD.A.19-05`, `EVT.A.19-07~08`, `POLICY.A.19-04` |
| 서비스 | `ClaimCouponHandler`, [발급 Handler](../A_19_30-service/issuance-handlers.md) |

## 책임과 경계

- `source_type=claim`인 `CouponIssueRequest` 하나를 접수한다.
- 이 요청 안에서 `UserCoupon`을 직접 만들거나 발급 수량을 확정하지 않는다.
- 사용자 자격 원본은 사용자 Context가 소유하며 쿠폰은 검증 SnapshotRef만 보존한다.

## 보안과 개인정보

- `user_id`를 body로 받지 않고 검증된 Principal에서 가져온다.
- 사용자 등급·차단 상태 원문과 프로필을 응답·로그에 남기지 않는다.
- 캠페인 존재와 발급 불가 내부 사유는 공개 오류 정책에 따라 최소화한다.

## 처리 규칙

1. 캠페인 기간, 승인 상태, 운영 중지와 사용자 자격 Snapshot을 확인한다.
2. `ClaimCouponHandler`가 업무 고유키로 발급 요청을 접수한다.
3. outbox Event가 수량 예약과 비동기 발급 Policy를 시작한다.
4. `issueRequestId`와 상태 조회 위치를 `202 Accepted`로 반환한다.

## 상태 변경과 트랜잭션

- `CouponIssueRequest` 접수 상태, 발급 요청 원장과 outbox를 같은 트랜잭션에 저장한다.
- 수량 예약은 후속 `CMD.A.19-26`, 실제 발급은 `CMD.A.19-07`이 각 Aggregate에서 수행한다.
- Redis gate 성공만으로 접수 성공을 반환하지 않는다.

## 멱등성과 동시성

- 범위는 `API.A.19-01 + user_id + campaign_id + Idempotency-Key`다.
- 같은 key·같은 요청은 기존 `issueRequestId`를 반환하고 다른 payload는 충돌로 거절한다.
- 캠페인·사용자·source 업무 unique와 `issue_request_id`별 수량 전이가 중복 발급을 막는다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 기간·자격·수량·중지 조건 불충족 | 안정적인 업무 거절 code만 반환 | 최신 화면 상태를 조회한다. |
| 응답 유실 | 새 발급 요청을 만들지 않음 | 같은 key로 재요청한다. |
| 필수 Snapshot·저장소 장애 | 성공으로 간주하지 않음 | `Retry-After` 뒤 같은 key로 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ClaimCouponHandler` |
| Aggregate | `CouponIssueRequest` |
| Repository | `CouponIssueRequestRepository` |
| Port | `UserEligibilityPort`, 운영 중지 조회, Redis gate |
| Event | `EVT.A.19-07` 또는 `EVT.A.19-08` |

## 관측성과 운영

- 캠페인, 일반화된 결과, issueRequest와 지연만 기록한다.
- 사용자 ID와 자격 Snapshot은 metric label에 넣지 않는다.
- 캠페인·사용자 기준 rate limit은 Postgres 수량 원장을 대체하지 않는다.

## 검증 항목

- 같은 key 동시 요청이 발급 요청 하나만 만든다.
- 수량 소진·운영 중지·자격 확인 실패에서 부분 원장이 남지 않는다.
- `202` 응답의 상태 위치가 같은 발급 요청을 가리킨다.

## 연관 시퀀스

- 직접 연결된 `80-sequence` 문서는 아직 없다. 비동기 후속 처리는 발급 Handler 문서를 따른다.

## 호환성과 변경 정책

- 선택적 표시 metadata 추가는 하위 호환이다. 동기 발급 완료를 보장하는 변경은 새 계약이 필요하다.

## 확인 필요

- `HOTSPOT.A.19-01`: 접수와 실제 발급 완료를 UI에서 표현하는 방식.
