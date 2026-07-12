---
id: API.A.19-11
title: 쿠폰 발급 조건 설정 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, campaign, quota]
source: local
created: 2026-07-11
updated: 2026-07-12
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-11 쿠폰 발급 조건 설정

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `PUT /api/v1/internal/coupon-campaigns/{campaignId}/issuance-policy` |
| operationId | `configureCouponCampaignIssuance` |
| 역할 | 총수량, 사용자별 제한, 오픈·종료 조건을 설정한다. |
| API 유형 | Command |
| 인증·권한 | 캠페인 관리 workload와 소유·운영 권한 |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key`와 `expectedVersion` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1` |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_11_configure_campaign_issuance.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-002`, `UC.A.19-10` |
| BC | `CMD.A.19-02`, `RULE.A.19-01` |
| 서비스 | `ConfigureFirstComeLimitHandler`, [발급 Handler](../A_19_30-service/issuance-handlers.md) |

## 책임과 경계

- `CouponCampaign`의 발급 수량·기간 정책만 변경한다.
- 실제 발급 수량은 `issue_request_id`별 예약·확정·해제 원장이 관리한다.
- Redis 준비 상태를 정책 원장으로 사용하지 않는다.

## 보안과 개인정보

- 캠페인 소유 또는 운영 권한과 expected version을 검증한다.
- 사용자 대상 원본이나 사용자별 발급 기록을 입력으로 받지 않는다.

## 처리 규칙

1. 총수량, 사용자별 제한과 시간 구간을 검증한다.
2. 현재 캠페인 version과 변경 가능한 상태를 확인한다.
3. 정책과 outbox를 저장하고 Redis gate는 Event 뒤 재구성한다.

## 상태 변경과 트랜잭션

- `CouponCampaign`, 정책 원장과 outbox를 같은 트랜잭션에 저장한다.
- Redis seed 실패가 Postgres 정책을 되돌리지 않는다.

## 멱등성과 동시성

- 범위는 `campaign_id + expectedVersion + Idempotency-Key`다.
- version 조건부 갱신으로 병렬 설정의 lost update를 막는다.
- 같은 key·같은 payload는 현재 정책 version을 반환한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| version 불일치 | 현재 version만 제공 | 최신 캠페인을 조회해 다시 편집한다. |
| 이미 발급된 수량보다 작은 총량 | 업무 거절 | 정책 값을 조정한다. |
| Redis 장애 | 정책 저장 결과와 분리 | gate 재구성 Worker를 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ConfigureFirstComeLimitHandler` |
| Aggregate | `CouponCampaign` |
| Repository | `CouponCampaignRepository` |
| Adapter | Postgres 원장, Redis gate 보조 |
| Event | 캠페인 정책 갱신 Event |

## 관측성과 운영

- version, 결과, 수량 정책 변경과 gate 재구성 지연을 기록한다.

## 검증 항목

- 수량 예약 중 병렬 변경이 원장 불변조건을 깨지 않는다.
- Redis key 유실 뒤 Postgres 정책으로 재구성된다.
- 과거 version 요청이 현재 정책을 덮어쓰지 않는다.

## 연관 시퀀스

- 캠페인 등록 뒤 승인·오픈 전에 호출한다.

## 호환성과 변경 정책

- 수량 정책 필드 의미 변경은 새 policy version을 사용한다.

## 결정 반영

- `HOTSPOT.A.19-06`: 이 API의 발급 조건과 대량 작업의 `evaluationAsOf` 스냅샷을 구분한다. 대량 작업은 발급 직전에 차단·캠페인 종료·운영 중지만 다시 확인한다.
