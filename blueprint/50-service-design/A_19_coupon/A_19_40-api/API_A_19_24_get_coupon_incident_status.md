---
id: API.A.19-24
title: 쿠폰 장애 현황 조회 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, incident, observability, query]
source: local
created: 2026-07-11
updated: 2026-07-11
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-24 쿠폰 장애 현황 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/internal/coupon-incidents/status` |
| operationId | `getCouponIncidentStatus` |
| 역할 | 발급·사용·복구 업무 상태와 Redis·MQ·Worker 보조 지표를 조회한다. |
| API 유형 | Query |
| 인증·권한 | 온콜·운영 workload |
| 노출 범위 | internal |
| 멱등성 | 해당 없음 |
| 캐시 | `no-store` |
| 호환성 | 보조 지표는 optional |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_24_get_coupon_incident_status.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.NFR-013`, `NFR-015~016`, `UC.A.19-21` |
| Read Model | `RM.A.19-07` |
| 서비스 | [이벤트 처리](../A_19_30-service/event-processing.md), [운영 Worker](../A_19_30-service/operations-workers.md) |

## 책임과 경계

- 발급 요청·실제 발급·최종 실패, 예약·확정·해제·회수와 복구 backlog를 기준 시각과 함께 제공한다.
- Redis, MQ, Worker 지표는 보조 신호이며 업무 성공 원장을 대체하지 않는다.
- 인시던트 원본과 경보 승인 상태는 외부 관측·운영 시스템이 소유한다.

## 보안과 개인정보

- 온콜·운영 권한을 확인하고 사용자·주문·코드 식별자를 반환하지 않는다.
- 외부 지표 query와 내부 endpoint 주소를 응답에 노출하지 않는다.

## 처리 규칙

1. 조회 범위와 기준 시간 구간을 검증한다.
2. Postgres 업무 Read Model을 먼저 조회한다.
3. 사용 가능한 외부 관측 지표를 보조로 결합하고 unavailable 상태를 명시한다.

## 상태 변경과 트랜잭션

- 상태를 변경하지 않는다.
- 외부 관측 장애를 업무 지표 0으로 바꾸지 않는다.

## 멱등성과 동시성

- Query이므로 멱등 키가 필요 없다.
- 각 source의 `asOf`를 별도로 반환해 시점 차이를 숨기지 않는다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| 보조 지표 장애 | `unavailable`로 표시 | 업무 원장 기준으로 판단한다. |
| Postgres Read Model 장애 | 정상 상태로 위조하지 않음 | 장애 응답 뒤 복구를 기다린다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Service | 쿠폰 장애 현황 Query |
| Read Model | `CouponIncidentStatus` |
| Port | `ObservabilityPort` |
| 원천 | 발급·사용·복구 Event, Redis·MQ·Worker 지표 참조 |

## 관측성과 운영

- 이 Endpoint 자체의 latency, source availability와 staleness를 기록한다.

## 검증 항목

- Redis 정상만으로 발급 정상 상태를 반환하지 않는다.
- 외부 지표 장애가 0값으로 보이지 않는다.
- 고카디널리티 내부 식별자가 응답·metric label에 없다.

## 연관 시퀀스

- 장애 확인 뒤 `API.A.19-17`, `19~21`로 운영 조치를 이어 갈 수 있다.

## 호환성과 변경 정책

- 새 보조 signal은 optional source로 추가하고 업무 지표의 의미를 바꾸지 않는다.

## 확인 필요

- 없음. 인시던트 승인·경보 임계값은 외부 운영 시스템 책임이다.
