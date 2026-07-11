---
id: API.A.19-19
title: 쿠폰 이벤트 복구 목록 조회 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, recovery, query]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-19 쿠폰 이벤트 복구 목록 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/internal/coupon-event-recoveries` |
| operationId | `listCouponEventRecoveries` |
| 역할 | 사용 이벤트 처리 실패와 재처리 상태를 cursor로 조회한다. |
| API 유형 | Query |
| 인증·권한 | 운영·온콜 workload와 실패 조회 권한 |
| 노출 범위 | internal |
| 멱등성 | 해당 없음 |
| 캐시 | `no-store` |
| 호환성 | cursor pagination |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_19_list_coupon_event_recoveries.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-015`, `UC.A.19-16` |
| Read Model | `RM.A.19-05` |
| 서비스 | [운영 Worker](../A_19_30-service/operations-workers.md), [이벤트 처리](../A_19_30-service/event-processing.md) |

## 책임과 경계

- recovery 상태, 원본 Command 유형, `payload_ref`, 시도 수와 다음 처리 시각을 제공한다.
- 원본 payload, 사용자 프로필과 주문·결제 상세를 반환하지 않는다.
- broker DLQ 자체를 원장으로 사용하지 않고 Postgres 복구 기록을 조회한다.

## 보안과 개인정보

- 실패 조회 권한과 운영 범위를 확인한다.
- `payload_ref`는 권한 있는 외부 시스템에서만 해석하며 이 응답에서 역참조하지 않는다.

## 처리 규칙

1. status, command type, time range, cursor와 limit를 검증한다.
2. `CouponEventRecovery` Read Model을 다음 처리 시각·ID 순으로 조회한다.
3. 상태, attempt 요약, 업무 결과 ref와 `asOf`를 반환한다.

## 상태 변경과 트랜잭션

- 상태를 변경하지 않는다.
- 조회 자체가 retry lease나 next_attempt_at을 바꾸지 않는다.

## 멱등성과 동시성

- Query이므로 멱등 키가 필요 없다.
- cursor에 정렬 시각과 recovery ID를 포함해 중복·누락을 막는다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| cursor 위변조·만료 | 내부 키를 숨김 | 첫 페이지부터 다시 조회한다. |
| projection 장애 | 빈 목록으로 위조하지 않음 | 저장소 복구 뒤 재조회한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Service | 복구 목록 Query |
| Aggregate·Read Model | `CouponEventRecovery`, `CouponFailureEvent` |
| Repository | 복구 Query Repository |

## 관측성과 운영

- filter, 결과 수, 오래된 next_attempt_at 수와 projection lag를 기록한다.

## 검증 항목

- payload 원문과 민감 식별자가 응답·로그에 노출되지 않는다.
- 같은 실패 Event가 복구 항목을 중복 생성하지 않는다.
- cursor 경계에서 항목이 누락되지 않는다.

## 연관 시퀀스

- 재처리는 `API.A.19-20`, 종료는 `API.A.19-21`로 이어진다.

## 호환성과 변경 정책

- 새 recovery status는 unknown 처리 정책과 함께 추가한다.

## 확인 필요

- `HOTSPOT.A.19-06`: 자동 재처리 횟수·간격·종료 기준.
