---
id: SD.A.1920.03
title: Context 쿠폰 조회 모델과 인덱스 설계
type: service-design-persistence
status: draft
tags: [service-design, coupon, read-model, index, redis, migration]
source: local
created: 2026-07-10
updated: 2026-07-12
service_design: SD.A.19
bounded_context: BC.A.19
domain_model: SD.A.1910
persistence: SD.A.1920
---

# Context 쿠폰 조회 모델과 인덱스 설계

## 책임

BC의 9개 Read Model 투영, 쓰기 모델의 `UNIQUE`·partial `UNIQUE`·조회 인덱스, Redis/MQ와 Postgres 사이의 정합성, 보존과 단계적 마이그레이션 원칙을 정의한다.

## 연관 문서

- 원천: [BC.A.19](../../../40-event-storming-bounded-context/BC_A_19_coupon.md), [REQ.A.02](../../../00-requirements/REQ_A_02_coupon_benefit.md)
- 결정: [Context 쿠폰 Hotspot 결정 기록](../hotspot-decisions.md)
- 도메인: [공통 계약](../A_19_10-domain-model/shared-contracts.md)
- 저장: [쓰기 모델](write-models.md), [원장과 신뢰성](ledgers-and-reliability.md)
- 서비스: [이벤트 처리](../A_19_30-service/event-processing.md)

## 조회 투영

| Read Model | 물리 모델 | 투영 원천 | 기본 조회 키 |
| --- | --- | --- | --- |
| `RM.A.19-01` 구매자 쿠폰함 | `rm_user_coupon_wallet` | 발급 요청, 사용자 쿠폰, 사용 Event | `(user_id, display_status, expires_at)` |
| `RM.A.19-02` 쿠폰 상세·적용 조건 | `rm_coupon_details` | 캠페인 정책 버전, 사용자 쿠폰 | `(campaign_id, policy_version)`, `user_coupon_id` |
| `RM.A.19-03` 판매자 쿠폰 성과 | `rm_seller_coupon_performance_daily` | 캠페인, 발급, 사용·회수 Event | `(seller_ref, business_date, campaign_id)` |
| `RM.A.19-04` 쿠폰 발급·사용 성과 | `rm_coupon_performance_minutely` | 대량 작업, 발급, 사용 Event | `(campaign_id, bucket_at)` |
| `RM.A.19-05` 쿠폰 실패 이벤트 | `rm_coupon_failures` | 발급 실패, 사용 복구 Event | `(failure_status, next_attempt_at)`, `business_key` |
| `RM.A.19-06` 사용자 쿠폰 타임라인 | `rm_user_coupon_timeline` | 발급·사용·만료 원장 | `(user_id, occurred_at desc)` |
| `RM.A.19-07` 쿠폰 장애 현황 | `rm_coupon_incident_status` | 업무 집계 + 외부 관측 지표 참조 | `(scope_type, scope_ref, observed_at)` |
| `RM.A.19-08` 쿠폰 비용 귀속 | `rm_coupon_cost_attribution` | 사용 확정·회수·비용 Event | `(order_id, redemption_id)`, `settlement_ref` |
| `RM.A.19-09` 쿠폰 읽기 전용 안내 | `rm_coupon_read_only_notice` | 운영 제어 Event | `(scope_type, scope_ref, effective_from)` |

쿠폰함의 `display_status`는 `CouponIssueRequest`, `UserCoupon`, `CouponRedemption` Event를 합성한다. 쓰기 Aggregate에 화면 전용 상태를 역기록하지 않는다. 투영 행은 `last_event_id`와 `projection_version`을 가져 재처리와 재구축을 지원한다.

### `RM.A.19-03` 판매자 쿠폰 성과

- 조회 주체의 `seller_ref`가 캠페인의 소유자 또는 비용 부담 주체와 일치하는 행만 투영·조회한다.
- 집계 차원은 `campaign_id`, 선택적 `drop_ref`, 선택적 `product_ref`, `business_date`이며 응답에 `asOf`를 포함한다.
- 발급, 사용, 취소·회수 집계와 판매자 부담 할인 금액만 제공한다. 공동 부담인 경우에도 플랫폼 부담액은 판매자 응답에 포함하지 않는다.
- `user_id`, `user_coupon_id`, 쿠폰 코드, `order_id`와 주문 상세는 저장하거나 응답하지 않는다.
- 마케팅·운영 전체 집계인 `RM.A.19-04`와 물리 투영을 분리해 판매자 권한이 운영 범위로 확대되지 않게 한다.

## Unique와 partial unique

| 저장 모델 | 제약 | 보호하는 규칙 |
| --- | --- | --- |
| `coupon_quantity_reservations` | `UNIQUE(campaign_id, issue_request_id)` | `RULE.A.19-01` 수량 전이 멱등성 |
| `coupon_issue_requests` | `UNIQUE(campaign_id, user_id, business_key)` | `RULE.A.19-02` 발급 멱등성 |
| `user_coupons` | `UNIQUE(issue_request_id)` | 실제 발급 결과 하나 |
| `coupon_codes` | `UNIQUE(code_hash)` | 코드 중복 방지 |
| `coupon_codes` | `UNIQUE(reserved_issue_request_id) WHERE status = 'reserved'` | 요청 하나의 활성 코드 예약 |
| `coupon_redemptions` | `UNIQUE(user_coupon_id) WHERE status = 'reserved'` | `RULE.A.19-03` 한 쿠폰 한 활성 주문 |
| `coupon_redemptions` | `UNIQUE(order_id, user_coupon_id, business_key)` | `RULE.A.19-04` 사용 처리 멱등성 |
| `coupon_recovery_attempts` | `UNIQUE(recovery_id, attempt_id, business_key)` | `RULE.A.19-05` 복구 상관관계 |
| `coupon_recovery_attempts` | `UNIQUE(recovery_id) WHERE status = 'retrying'` | 한 복구 기록의 활성 시도 하나 |
| `consumer_inbox` | `UNIQUE(consumer_name, event_id)` | Event 중복 소비 방지 |

## 조회 인덱스

| 테이블 | 인덱스 열 | 목적 |
| --- | --- | --- |
| `coupon_campaigns` | `(status, starts_at, ends_at)` | 활성·예정 캠페인 조회 |
| `coupon_applicability_policies` | `(campaign_id, policy_version, target_type, target_ref)` | 주문 스냅샷 적용 조건 조회 |
| `coupon_codes` | `(code_hash) INCLUDE (status, campaign_id, reserved_until)` | 코드 등록 단건 검증 |
| `coupon_issue_requests` | `(status, next_attempt_at, issue_request_id)` | 발급 Worker 점유 |
| `coupon_issue_requests` | `(user_id, created_at desc)` | 쿠폰함·CS 타임라인 투영 |
| `user_coupons` | `(user_id, status, expires_at)` | 보유·만료 대상 조회 |
| `coupon_redemptions` | `(order_id, status)`, `(user_coupon_id, created_at desc)` | 주문 사용 처리와 쿠폰 이력 |
| `bulk_coupon_issue_jobs` | `(status, created_at)` | 실행 대기 작업 점유 |
| `coupon_operational_scopes` | `(scope_type, scope_ref, control_id)` | 중지·안내 범위 확인 |
| `coupon_event_recoveries` | `(status, next_attempt_at, recovery_id)` | 복구 Worker 점유 |
| `domain_outbox` | `(publish_status, next_attempt_at, occurred_at)` | 발행 대기 Event 점유 |

Worker 점유 조회는 `FOR UPDATE SKIP LOCKED` 또는 같은 효과를 가진 방식을 사용한다. 인덱스는 실제 분포와 실행 계획을 확인해 조정하되, unique 제약은 성능 최적화와 분리해 유지한다.

## Redis 정합성

| Redis 자료 | 용도 | Postgres 기준 | 장애 시 처리 |
| --- | --- | --- | --- |
| 캠페인 잔여 수량 gate | 명백한 초과 요청 선행 차단 | 수량 예약·확정·해제 원장 | 신규 예약 제한 또는 재구축; Redis 값으로 원장 수정 금지 |
| 활성 정책 캐시 | 주문 검증 읽기 가속 | 정책 버전 쓰기 모델 | 버전 불일치면 Postgres 조회 후 교체 |
| 운영 중지·안내 캐시 | 빠른 범위 판단 | `CouponOperationalControl` | 캐시에서 확인할 수 없으면 Postgres에서 판단하며 성공으로 간주하는 기본값은 사용하지 않는다. |
| 쿠폰함 캐시 | 반복 조회 가속 | `rm_user_coupon_wallet` | projection version 불일치면 폐기 |

캐시 키는 `schema_version`과 정책·투영 버전을 포함한다. TTL은 유일한 무효화 수단이 아니며 outbox Event로 삭제 또는 갱신한다.

## MQ 정합성

- MQ 적재 성공은 발급 성공이 아니라 outbox 전달 성공이다.
- 소비자는 `event_id` inbox와 업무 고유키를 모두 확인한다.
- 순서가 뒤바뀐 Event는 Aggregate 버전과 현재 상태로 판정하고, 아직 적용할 수 없으면 재처리 가능 실패로 보존한다.
- DLQ는 원본 업무 상태가 아니다. `CouponIssueRequest` 또는 `CouponEventRecovery`가 업무 실패와 복구 상태를 소유한다.
- 큐 적재량, 소비 지연, 브로커 상태는 `RM.A.19-07`에 참조되지만 Postgres 상태를 덮어쓰지 않는다.

## 보존

| 데이터 | 보존 원칙 |
| --- | --- |
| 캠페인·사용자 쿠폰·사용 원장·비용 귀속 | 감사·정산 요구 기간 동안 원본 식별자와 금액을 보존한다. 정확한 기간은 조직 정책으로 결정한다. |
| 발급·수량·복구 원장 | 관련 Aggregate와 결과 참조보다 짧게 보존하지 않는다. |
| outbox/inbox | 재처리·감사 기간이 지난 뒤 `payload`를 축약할 수 있으나 `event_id`, 유형, 결과 상태는 남긴다. |
| Read Model | 재구축 가능하므로 쓰기 원장보다 짧게 보존할 수 있다. |
| 외부 스냅샷 | 판단 재현에 필요한 최소 필드, 버전, 해시만 보존한다. |
| 사용자 식별자 | 접근 통제·가명화 정책을 적용하되 정산·분쟁 근거와 충돌하지 않게 별도 절차로 처리한다. |

## 단계적 마이그레이션

1. 새 쓰기 테이블과 제약, 원장, outbox/inbox를 추가한다.
2. 기존 데이터가 있으면 원본별 변환 규칙과 검증 합계를 만든 뒤 backfill한다.
3. Event 투영을 과거 원장부터 재생해 9개 Read Model을 구축한다.
4. shadow read로 기존 조회와 새 투영의 결과·카운터를 비교한다.
5. 발급 수량, 사용자별 발급, 활성 사용 예약, 비용 합계 대사를 통과한 뒤 읽기 경로를 전환한다.
6. 쓰기 전환 뒤 일정 기간 이중 대사를 유지한다. 이중 쓰기는 원장 불일치 원인이 되므로 허용하지 않는다.
7. 기존 저장소 제거는 별도 승인과 rollback 기준을 마련한 뒤 진행한다.
