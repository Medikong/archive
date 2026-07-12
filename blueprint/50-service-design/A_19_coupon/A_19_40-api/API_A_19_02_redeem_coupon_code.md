---
id: API.A.19-02
title: 쿠폰 코드 등록 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, code, issuance]
source: local
created: 2026-07-11
updated: 2026-07-12
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-02 쿠폰 코드 등록

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/coupon-code-redemptions` |
| operationId | `redeemCouponCode` |
| 역할 | 코드 원문을 검증해 발급 처리 동안 예약한다. |
| API 유형 | Command |
| 인증·권한 | 사용자 Principal의 본인 등록 |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `private, no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_02_redeem_coupon_code.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-025`, `UC.A.19-05` |
| BC | `CMD.A.19-06`, `EVT.A.19-12~15`, `POLICY.A.19-05` |
| 서비스 | `RedeemCouponCodeHandler`, [발급 Handler](../A_19_30-service/issuance-handlers.md) |

## 책임과 경계

- `CouponCodeBatch` 안의 코드 상태를 하나의 `issue_request_id`에 예약한다.
- 발급 요청 생성, 사용자 쿠폰 생성과 코드 사용 완료는 후속 Event·Policy가 각각 처리한다.
- 코드 원문을 조회 API나 Event로 다시 내보내지 않는다.

## 보안과 개인정보

- 코드 원문은 입력 단계에서만 사용하고 정규화한 뒤 keyed hash로 조회한다.
- 원문, 전체 hash와 사용자 ID를 로그·trace·metric에 남기지 않는다.
- 존재하지 않음, 만료, 대상 아님을 악용할 수 있는 상세 내부 정보는 제한한다.

## 처리 규칙

1. 코드 형식과 공개 rate limit을 확인한다.
2. 상태·기간·대상·중복 등록을 검증하고 `issue_request_id`에 예약한다.
3. 예약 Event가 `CMD.A.19-13`을 요청하고 비동기 발급을 시작한다.
4. 발급 요청과 상태 조회 위치를 `202 Accepted`로 반환한다.

## 상태 변경과 트랜잭션

- 코드 예약 상태, 코드 원장과 outbox를 같은 트랜잭션에 저장한다.
- 발급 성공 뒤 `CMD.A.19-16`, 최종 실패 뒤 `CMD.A.19-17`이 같은 예약을 닫는다.
- `UserCoupon`과 `CouponIssueRequest`를 이 트랜잭션에 함께 쓰지 않는다.

## 멱등성과 동시성

- 범위는 `API.A.19-02 + user_id + code_fingerprint + Idempotency-Key`다.
- 같은 코드의 동시 등록은 partial unique와 row lock으로 한 예약만 성공한다.
- 같은 key의 replay는 기존 issueRequest를 반환하며 코드 원문을 저장하지 않는다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 형식·상태·대상 조건 불충족 | 허용된 코드 등록 결과만 공개 | 입력을 확인하거나 지원 절차를 따른다. |
| 발급 최종 실패 | 코드 예약을 Event·Policy로 해제 | 발급 요청 상태를 조회한 뒤 재시도한다. |
| 응답 유실 | 코드 예약을 중복 생성하지 않음 | 같은 key로 재요청한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `RedeemCouponCodeHandler` |
| Aggregate | `CouponCodeBatch` |
| Repository | `CouponCodeBatchRepository` |
| Port | 사용자 자격 Snapshot, rate limit |
| Event | `EVT.A.19-12~15` |

## 관측성과 운영

- 정규화 결과가 아닌 성공·거절 유형과 지연만 관측한다.
- 사용자·IP rate limit은 공격 완화 수단이며 코드 상태 원장을 대신하지 않는다.

## 검증 항목

- 같은 코드 동시 요청에서 예약 하나만 남는다.
- 원문이 DB 일반 컬럼, Event, 로그와 trace에 남지 않는다.
- 발급 성공·최종 실패 후 코드 예약이 각각 확정·해제된다.

## 연관 시퀀스

- 직접 연결된 `80-sequence` 문서는 아직 없다.

## 호환성과 변경 정책

- 코드 정규화 변경은 기존 hash lookup 호환성과 version 전략을 먼저 정의해야 한다.

## 결정 반영

- `HOTSPOT.A.19-01`: 코드 검증·예약 뒤에는 `발급 대기`를 반환하고 실제 사용자 쿠폰 생성과 수량 확정 뒤에만 `발급 완료`로 표시한다.
