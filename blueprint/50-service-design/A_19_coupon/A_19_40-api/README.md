---
id: SD.A.1940
title: Context 쿠폰 API 설계
type: service-design-api
status: draft
tags: [service-design, coupon, api, rest, openapi, event-contract]
source: local
created: 2026-07-09
updated: 2026-07-11
service_design: SD.A.19
bounded_context: BC.A.19
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# Context 쿠폰 API 설계

## 역할

Context 쿠폰의 공개 REST API, 내부 workload API, 공통 HTTP 계약과 MQ Event 경계를 정의한다. HTTP wire 계약은 OpenAPI를 원장으로 사용하고, 책임·트랜잭션·멱등성·복구 판단은 Endpoint별 Markdown에서 관리한다.

## 원천

- [BC.A.19 Context 쿠폰](../../../40-event-storming-bounded-context/BC_A_19_coupon.md)
- [REQ.A.02 쿠폰 및 혜택](../../../00-requirements/REQ_A_02_coupon_benefit.md)
- [UC.A.19 쿠폰 사용자 목표](../../../30-uc/UC_A_19_coupon_wallet.md)
- [SD.A.1910 도메인 모델](../A_19_10-domain-model/README.md)
- [SD.A.1920 영속성](../A_19_20-persistence/README.md)
- [SD.A.1930 서비스](../A_19_30-service/README.md)

## 계약 원장

| 문서 | 책임 |
| --- | --- |
| [OpenAPI 진입 문서](openapi/openapi.yaml) | `/api/v1` Path, tag, security scheme 등록 |
| [Path Item](openapi/paths/) | Method/Path, parameter, 요청·응답 schema, HTTP 상태, 오류와 예시 |
| [공통 component](openapi/components/) | 식별자, 업무 schema, header, parameter, 오류 응답과 인증 방식 |
| Endpoint별 Markdown | 책임 경계, Handler, Aggregate, 트랜잭션, 멱등성과 복구 판단 |
| [Event 계약](event-contracts.md) | 41개 Domain Event의 공통 envelope, outbox/inbox와 외부 수신 경계 |

## 통신 경계

| 경계 | 프로토콜 | 인증·신뢰 근거 | 계약 상태 |
| --- | --- | --- | --- |
| 구매자 앱·BFF → 쿠폰 | HTTPS REST/JSON | 웹 Session+CSRF+Origin 또는 모바일 Bearer Principal | OpenAPI 확정 |
| 주문·결제 → 쿠폰 | 내부 HTTPS REST/JSON | workload credential, 사용자·주문 SnapshotRef | OpenAPI 확정 |
| 판매자·운영·CS·정산 업무 시스템 → 쿠폰 | 내부 HTTPS REST/JSON | workload credential, 권한, 위험 작업의 approvalRef | OpenAPI 확정 |
| 쿠폰 ↔ MQ | versioned JSON Event | outbox/inbox, event_id, schema version | [Event 계약](event-contracts.md) |
| 서비스 간 gRPC | 사용하지 않음 | 현재 BC·REQ에 gRPC 계약이 없음 | 미채택 |

## 공통 HTTP 계약

- Base Path는 `/api/v1`이다. `/internal` Path는 외부 Gateway에 공개하지 않는다.
- 성공 응답은 `data`와 `meta.requestId`를 분리한다. 비동기 접수는 `202 Accepted`, 상태 조회 `Location`과 `Retry-After`를 제공한다.
- 오류는 `application/problem+json`과 안정적인 `COUPON_*` code를 사용한다. 내부 ID, 코드 원문, 외부 payload 원문과 승인 원문은 공개하지 않는다.
- 모든 요청은 `X-Request-Id`와 W3C `traceparent`를 지원한다. Command는 `Idempotency-Key`를 필수로 받는다.
- 멱등 범위는 `API ID + actor/workload + 업무 소유자 + Idempotency-Key`다. 같은 key의 다른 canonical payload는 `COUPON_IDEMPOTENCY_CONFLICT`로 거절한다.
- 사용자별 응답, 운영·CS·정산 응답과 코드 처리 응답은 `Cache-Control: private, no-store`다. 성과·장애 집계도 권한별 범위가 달라 공유 캐시를 사용하지 않는다.
- 외부 참조는 `ExternalRef`와 `SnapshotRef`만 받는다. 사용자·상품·드롭·주문·결제·CS·정산 원본을 요청 body에 복제하지 않는다.
- 한 HTTP Command는 하나의 Aggregate Handler만 직접 호출한다. 후속 Aggregate 변경은 outbox Event와 Policy로 이어진다.
- Redis와 MQ 결과만으로 성공을 반환하지 않는다. Postgres 원장·제약과 idempotency 결과가 최종 판단 근거다.

## 공개 구매자 API

| API ID | Method / Path | 책임 | Command·Read Model | 설계 노트 |
| --- | --- | --- | --- | --- |
| `API.A.19-01` | `POST /api/v1/coupon-campaigns/{campaignId}/claims` | 쿠폰 직접 수령 접수 | `CMD.A.19-05` | [쿠폰 수령](API_A_19_01_claim_coupon.md) |
| `API.A.19-02` | `POST /api/v1/coupon-code-redemptions` | 코드 검증·예약과 발급 접수 | `CMD.A.19-06` | [쿠폰 코드 등록](API_A_19_02_redeem_coupon_code.md) |
| `API.A.19-03` | `GET /api/v1/users/me/coupons` | 상태별 쿠폰함과 활성 안내 조회 | `RM.A.19-01`, `RM.A.19-09` | [내 쿠폰 목록](API_A_19_03_list_my_coupons.md) |
| `API.A.19-04` | `GET /api/v1/users/me/coupons/{userCouponId}` | 내 쿠폰 상세·적용 조건 조회 | `RM.A.19-02` | [내 쿠폰 상세](API_A_19_04_get_my_coupon.md) |

## 주문·결제 내부 API

| API ID | Method / Path | 책임 | Command | 설계 노트 |
| --- | --- | --- | --- | --- |
| `API.A.19-05` | `POST /api/v1/internal/coupon-validations` | 조건·할인·비용 Snapshot 평가 | `CMD.A.19-09` | [쿠폰 검증](API_A_19_05_validate_coupon.md) |
| `API.A.19-06` | `POST /api/v1/internal/coupon-redemptions/{redemptionId}/reserve` | 검증된 주문별 사용 예약 | `CMD.A.19-10` | [사용 예약](API_A_19_06_reserve_coupon_redemption.md) |
| `API.A.19-07` | `POST /api/v1/internal/coupon-redemptions/{redemptionId}/commit` | 예약 사용 확정 | `CMD.A.19-11` | [사용 확정](API_A_19_07_commit_coupon_redemption.md) |
| `API.A.19-08` | `POST /api/v1/internal/coupon-redemptions/{redemptionId}/release` | 확정 전 예약 해제 | `CMD.A.19-12` | [예약 해제](API_A_19_08_release_coupon_redemption.md) |
| `API.A.19-09` | `POST /api/v1/internal/coupon-redemptions/{redemptionId}/revoke` | 확정 사용 회수·비용 보정 | `CMD.A.19-15` | [사용 회수](API_A_19_09_revoke_coupon_redemption.md) |

## 캠페인·운영 API

| API ID | Method / Path | 책임 | Command·Read Model | 설계 노트 |
| --- | --- | --- | --- | --- |
| `API.A.19-10` | `POST /api/v1/internal/coupon-campaigns` | 캠페인 정책 등록 | `CMD.A.19-01` | [캠페인 등록](API_A_19_10_create_coupon_campaign.md) |
| `API.A.19-11` | `PUT /api/v1/internal/coupon-campaigns/{campaignId}/issuance-policy` | 선착순 수량·기간 설정 | `CMD.A.19-02` | [발급 조건 설정](API_A_19_11_configure_campaign_issuance.md) |
| `API.A.19-12` | `POST /api/v1/internal/coupon-campaigns/{campaignId}/reviews` | 판매자·제휴 쿠폰 검토 | `CMD.A.19-03` | [캠페인 검토](API_A_19_12_review_coupon_campaign.md) |
| `API.A.19-13` | `POST /api/v1/internal/coupon-campaigns/{campaignId}/policy-versions` | 정책 새 버전 등록 | `CMD.A.19-04` | [정책 변경](API_A_19_13_change_coupon_campaign_policy.md) |
| `API.A.19-14` | `POST /api/v1/internal/bulk-coupon-issue-jobs` | 대량 발급 작업 접수 | `CMD.A.19-08` | [대량 작업 등록](API_A_19_14_create_bulk_coupon_issue_job.md) |
| `API.A.19-15` | `GET /api/v1/internal/bulk-coupon-issue-jobs/{bulkJobId}` | 대량 작업 진행·결과 조회 | `RM.A.19-04`, `RM.A.19-05` | [대량 작업 조회](API_A_19_15_get_bulk_coupon_issue_job.md) |
| `API.A.19-16` | `GET /api/v1/internal/coupon-campaigns/{campaignId}/performance` | 발급·사용 성과 조회 | `RM.A.19-04` | [캠페인 성과](API_A_19_16_get_coupon_campaign_performance.md) |
| `API.A.19-17` | `POST /api/v1/internal/coupon-operational-controls` | 승인된 발급·사용 중지 적용 | `CMD.A.19-20` | [운영 중지](API_A_19_17_apply_coupon_operational_control.md) |
| `API.A.19-18` | `PUT /api/v1/internal/coupon-operational-controls/{controlId}/read-only-notice` | 같은 범위의 안내 적용 | `CMD.A.19-31` | [읽기 전용 안내](API_A_19_18_apply_coupon_read_only_notice.md) |

## 복구·CS·정산 API

| API ID | Method / Path | 책임 | Command·Read Model | 설계 노트 |
| --- | --- | --- | --- | --- |
| `API.A.19-19` | `GET /api/v1/internal/coupon-event-recoveries` | 실패·재처리 대상 조회 | `RM.A.19-05` | [복구 목록](API_A_19_19_list_coupon_event_recoveries.md) |
| `API.A.19-20` | `POST /api/v1/internal/coupon-event-recoveries/{recoveryId}/retry-attempts` | 새 재처리 시도 요청 | `CMD.A.19-21` | [복구 재시도](API_A_19_20_retry_coupon_event_recovery.md) |
| `API.A.19-21` | `POST /api/v1/internal/coupon-event-recoveries/{recoveryId}/finalization` | 사용 복구 최종 실패 종료 | `CMD.A.19-25` | [복구 종료](API_A_19_21_finalize_coupon_event_recovery.md) |
| `API.A.19-22` | `GET /api/v1/internal/users/{userId}/coupon-timeline` | 승인된 사용자 쿠폰 이력 조회 | `RM.A.19-06` | [사용자 타임라인](API_A_19_22_get_user_coupon_timeline.md) |
| `API.A.19-23` | `POST /api/v1/internal/compensation-coupon-issue-requests` | 승인된 보상 발급 요청 생성 | `CMD.A.19-13` | [보상 발급 요청](API_A_19_23_create_compensation_coupon_issue_request.md) |
| `API.A.19-24` | `GET /api/v1/internal/coupon-incidents/status` | 업무 상태와 보조 기술 지표 조회 | `RM.A.19-07` | [장애 현황](API_A_19_24_get_coupon_incident_status.md) |
| `API.A.19-25` | `GET /api/v1/internal/coupon-cost-attributions` | 주문별 비용 귀속 조회 | `RM.A.19-08` | [비용 귀속](API_A_19_25_list_coupon_cost_attributions.md) |

## HTTP로 노출하지 않는 Command

`CMD.A.19-07`, `14`, `16~19`, `22~24`, `26~30`, `32~34`는 Policy·Worker·실패 처리기가 실행하는 내부 Command다. 외부 호출자가 이 전이를 직접 건너뛰지 못하도록 REST Endpoint를 만들지 않는다. 자동 지급은 `HOTSPOT.A.19-09`가 닫히기 전까지 외부 Event 계약을 확정하지 않는다.

## 공통 오류

| Code | HTTP | 의미 |
| --- | --- | --- |
| `COUPON_AUTHENTICATION_REQUIRED` | 401 | 검증된 사용자 또는 workload credential 없음 |
| `COUPON_FORBIDDEN` | 403 | 소유권·권한·승인 조건 불충족 |
| `COUPON_NOT_FOUND` | 404 | 공개 가능한 범위에서 대상 없음 |
| `COUPON_IDEMPOTENCY_CONFLICT` | 409 | 같은 key에 다른 canonical payload 사용 |
| `COUPON_VERSION_CONFLICT` | 409 | Aggregate 또는 정책 예상 version 불일치 |
| `COUPON_STATE_CONFLICT` | 409 | 현재 상태에서 요청한 전이를 수행할 수 없음 |
| `COUPON_OPERATION_STOPPED` | 409 | 승인된 운영 중지로 신규 발급·예약 차단 |
| `COUPON_BUSINESS_RULE_REJECTED` | 422 | 자격·기간·정책·할인·수량 규칙 불충족 |
| `COUPON_RATE_LIMITED` | 429 | 공개 수령·코드 등록 또는 운영 보호 한도 초과 |
| `COUPON_DEPENDENCY_UNAVAILABLE` | 503 | 필수 Snapshot·승인·저장소 확인 불가 |

## Hotspot 경계

| Hotspot | API에서 확정하지 않는 내용 |
| --- | --- |
| `HOTSPOT.A.19-01` | `202 accepted` 이후 UI의 발급 대기·완료 문구와 자동 이동 방식 |
| `HOTSPOT.A.19-02~03` | 주문 확정 원천 사건, 예약 해제 유예, 회수 뒤 재사용 여부 |
| `HOTSPOT.A.19-04` | 새 정책이 기존 사용자 쿠폰과 활성 예약에 미치는 영향 |
| `HOTSPOT.A.19-05` | 판매자·운영·CS 승인선, 보상 한도와 증빙 기준 |
| `HOTSPOT.A.19-06` | 대량 대상 평가 시점, 자동 재시도 횟수·간격·최종 실패 기준 |
| `HOTSPOT.A.19-07` | 판매자 성과 전용 Endpoint와 허용 집계 범위 |
| `HOTSPOT.A.19-08` | 여러 쿠폰의 조합 검증·예약 계약. 현재 Endpoint는 쿠폰 하나만 처리한다. |
| `HOTSPOT.A.19-09` | 자동 지급 Event 생산자, event type과 필수 `source_ref` schema |

## 검증

`archive` 저장소 루트에서 실행한다.

```bash
npx --yes @redocly/cli@2.38.0 lint \
  --config blueprint/50-service-design/A_19_coupon/A_19_40-api/openapi/redocly.yaml \
  blueprint/50-service-design/A_19_coupon/A_19_40-api/openapi/openapi.yaml

npx --yes @redocly/cli@2.38.0 bundle \
  blueprint/50-service-design/A_19_coupon/A_19_40-api/openapi/openapi.yaml \
  --output /tmp/A_19_coupon.openapi.bundle.yaml
```

bundle은 검증 생성물이며 직접 수정하지 않는다.
