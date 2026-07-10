---
id: SD.A.1920.02
title: Context 쿠폰 원장과 신뢰성 설계
type: service-design-persistence
status: draft
tags: [service-design, coupon, ledger, idempotency, outbox, inbox]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.19
bounded_context: BC.A.19
domain_model: SD.A.1910
persistence: SD.A.1920
---

# Context 쿠폰 원장과 신뢰성 설계

## 책임

발급 수량, 발급·사용 상태 변경, 멱등 처리, outbox/inbox와 이벤트 복구의 영속 기록과 트랜잭션 경계를 정의한다. Redis와 MQ는 이 기록의 가용성과 전달을 돕지만 최종 상태를 결정하지 않는다.

## 연관 문서

- 원천: [BC.A.19](../../../40-event-storming-bounded-context/BC_A_19_coupon.md), [REQ.A.02](../../../00-requirements/REQ_A_02_coupon_benefit.md)
- 도메인: [공통 계약](../A_19_10-domain-model/shared-contracts.md), [운영과 복구](../A_19_10-domain-model/operations-recovery.md)
- 저장: [쓰기 모델](write-models.md), [조회 모델과 인덱스](read-models-and-indexes.md)
- 서비스: [이벤트 처리](../A_19_30-service/event-processing.md), [운영 Worker](../A_19_30-service/operations-workers.md)

## append-only 원장

| 테이블 | 기록 단위 | 필수 열 |
| --- | --- | --- |
| `coupon_quantity_ledger` | 발급 수량 예약·확정·해제·거절 | `ledger_id`, `campaign_id`, `issue_request_id`, `transition`, `quantity`, `before_state`, `after_state`, `result_ref`, `occurred_at` |
| `coupon_issue_ledger` | 발급 요청 접수부터 완료·거절·최종 실패 | `ledger_id`, `issue_request_id`, `business_key`, `event_type`, `status`, `result_ref`, `failure_code`, `occurred_at` |
| `user_coupon_ledger` | 사용자 쿠폰 발급·만료·관리상 회수 | `ledger_id`, `user_coupon_id`, `issue_request_id`, `event_type`, `result_ref`, `occurred_at` |
| `coupon_redemption_ledger` | 검증·예약·확정·해제·회수·비용 귀속 | `ledger_id`, `redemption_id`, `order_id`, `user_coupon_id`, `event_type`, `amount_snapshot`, `result_ref`, `occurred_at` |
| `coupon_operation_ledger` | 운영 중지·안내 적용 | `ledger_id`, `control_id`, `scope`, `operation_request_ref`, `approval_ref`, `event_type`, `occurred_at` |
| `coupon_recovery_ledger` | 실패 기록, 재실행 대기·판정·최종 실패 | `ledger_id`, `recovery_id`, `attempt_id`, `business_key`, `event_type`, `result_ref`, `failure_code`, `occurred_at` |

원장 행은 수정·삭제하지 않는다. 잘못된 업무 결과는 기존 Event를 지우지 않고 별도 보정 Command와 Event로 남긴다.

## 발급 수량 전이

`coupon_quantity_reservations`의 primary key는 `(campaign_id, issue_request_id)`다. 예약 Command의 트랜잭션은 다음 순서를 지킨다.

1. 기존 예약 행을 조회한다. 종단 결과가 있으면 같은 `result_ref`를 반환한다.
2. 캠페인 행을 잠그거나 동일한 효과의 원자적 조건부 갱신을 수행한다.
3. `reserved_quantity + confirmed_quantity + requested_quantity <= total_quantity`를 검증한다.
4. 예약 행과 캠페인 집계, `coupon_quantity_ledger`, `domain_outbox`를 한 트랜잭션에 기록한다.
5. 확정은 예약 수를 줄이고 확정 수를 늘린다. 해제는 예약 수만 줄인다.

확정과 해제가 동시에 도착하면 예약 행의 현재 상태를 먼저 바꾼 요청만 성공한다. 나중 요청은 새 오류 결과를 만들지 않고 이미 확정되었거나 해제된 `result_ref`를 반환한다.

## 멱등성 레코드

`coupon_idempotency_records`는 전송 계층의 키가 아니라 업무 실행 결과를 재사용하기 위한 기록이다.

| 열 | 설명 |
| --- | --- |
| `operation_type`, `business_key` | 복합 primary key |
| `owner_type`, `owner_id` | Command 대상 Aggregate |
| `request_hash` | 같은 키에 다른 입력이 들어오는 것을 차단 |
| `status` | `processing`, `completed`, `failed_final` |
| `result_ref`, `response_snapshot` | 이미 처리된 도메인 결과 참조와 최소 응답 스냅샷 |
| `locked_until`, `completed_at`, `expires_at` | 점유와 보존 시각 |

- 같은 키와 같은 해시는 진행 중 또는 완료 결과를 재사용한다.
- 같은 키와 다른 해시는 충돌로 거절한다.
- `processing` 점유 만료만으로 업무 상태를 되돌리지 않는다. Aggregate와 원장을 확인해 재개한다.
- 수량·발급·사용·복구처럼 장기 감사가 필요한 업무 키는 결과 원장보다 먼저 삭제하지 않는다.

## Outbox

`domain_outbox`는 Aggregate 변경과 같은 Postgres 트랜잭션에서 기록한다.

| 열 | 설명 |
| --- | --- |
| `event_id` | Domain Event 전역 식별자이자 primary key |
| `event_type`, `payload_schema_version` | BC Event와 `payload` 버전 |
| `aggregate_type`, `aggregate_id`, `aggregate_version` | 생산 Aggregate와 순서 근거 |
| `correlation_id`, `causation_id`, `trace_id` | 연쇄·관측 상관관계 |
| `payload`, `occurred_at` | 외부 원본을 제외한 Event 본문과 발생 시각 |
| `publish_status`, `published_at`, `attempt_count`, `next_attempt_at` | 전달 상태 |

Outbox 발행기는 MQ 전송 성공 뒤 상태를 갱신한다. 전송과 상태 갱신 사이에서 장애가 나면 중복 전달될 수 있으므로 소비자는 inbox로 막는다. MQ의 순서는 Aggregate별 순서를 돕는 수단이며, 소비자는 `aggregate_version`과 현재 도메인 상태를 함께 확인한다.

## Inbox

`consumer_inbox`의 유일 키는 `(consumer_name, event_id)`다. 소비 트랜잭션은 inbox 삽입, Policy 결정 기록과 대상 Command 요청 저장을 함께 처리한다.

| 상태 | 의미 |
| --- | --- |
| `received` | Event를 처음 수신했다. |
| `processed` | Policy가 결과 또는 후속 Command 참조를 기록했다. |
| `failed_retryable` | 기술 오류로 재처리할 수 있다. |
| `failed_final` | 스키마 불일치 등 자동 재처리를 종료했다. |

도메인 실패를 MQ 재시도로 감추지 않는다. Command가 업무 거절 Event를 만들 수 있으면 inbox는 정상 처리로 닫고 해당 Event를 발행한다.

## 사용 이벤트 복구 상관관계

`coupon_event_recoveries`와 `coupon_recovery_attempts`는 다음 제약을 가진다.

- `(recovery_id, attempt_id, business_key)` unique
- 한 `recovery_id`에는 `retrying` 시도가 최대 하나인 partial unique
- 결과 갱신 조건은 `recovery_id`, 현재 `attempt_id`, `business_key`, 기존 상태 `retrying`을 모두 포함
- `transitioned`와 `already_applied`는 `result_ref` 필수
- `failed`는 `failure_code`와 재처리 가능 여부 필수

원본 `payload`는 복구 테이블에 복제하지 않고 변경할 수 없는 `original_payload_ref`와 해시를 둔다. 재실행 Worker는 참조가 가리키는 원본을 읽고 해시가 일치할 때만 실행한다.

## 트랜잭션 경계

| Command 책임 | 한 트랜잭션에 포함 | 후속 트랜잭션 |
| --- | --- | --- |
| 캠페인 수량 예약·확정·해제 | `CouponCampaign`, 수량 원장, 멱등 레코드, outbox | 발급 요청 거절·대기·완료 |
| 발급 요청 상태 전이 | `CouponIssueRequest`, 발급 원장, 멱등 레코드, outbox | 사용자 쿠폰 생성, 코드·대량 결과, 수량 결과 |
| 사용자 쿠폰 발급·만료 | `UserCoupon`, 사용자 쿠폰 원장, outbox | 발급 요청 완료, 수량 확정, 코드 확정, 예약 해제 |
| 사용 상태 전이 | `CouponRedemption`, 사용 원장, 멱등 레코드, outbox | 조회 투영, 정산·알림 전달, 실패 복구 기록 |
| 운영 제어 | `CouponOperationalControl`, 운영 원장, outbox | 캐시 무효화와 조회 투영 |
| 복구 기록·결과 반영 | `CouponEventRecovery`, 현재 attempt, 복구 원장, outbox | 원본 `CouponRedemption` 재실행 |

## 장애 후 대사

| 비교 | 불일치 처리 |
| --- | --- |
| 캠페인 수량 집계 ↔ 수량 원장 | 원장 합계로 경보를 만들고 승인된 보정 Command만 실행한다. |
| 발급 완료 요청 ↔ `UserCoupon` | 양방향 결과 참조를 확인하고 누락은 복구 대기열에 등록한다. |
| outbox 미발행 ↔ Aggregate Event | outbox 발행을 재시도한다. Aggregate 상태를 되돌리지 않는다. |
| MQ 전달 ↔ inbox | 중복은 기존 처리 결과를 반환하고 누락은 outbox부터 재전달한다. |
| Redis 수량 ↔ Postgres 수량 원장 | Redis 값을 최종 근거로 Postgres 원장을 수정하지 않고 캐시를 재구축한다. |
