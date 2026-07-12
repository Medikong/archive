---
id: SD.A.1910.05
title: Context 쿠폰 공통 도메인 계약
type: service-design-domain-contracts
status: draft
tags: [service-design, coupon, contracts, event, policy, read-model]
source: local
created: 2026-07-10
updated: 2026-07-12
service_design: SD.A.19
bounded_context: BC.A.19
domain_model: SD.A.1910
---

# Context 쿠폰 공통 도메인 계약

## 책임

책임별 도메인 문서가 함께 사용하는 식별자, Value Object, Event 메타데이터, Policy 실행 원칙, Business Rule과 Read Model의 경계를 정의한다. 개별 Aggregate의 필드와 상태 전이는 각 책임 문서에서 관리한다.

## 연관 문서

- 원천: [BC.A.19](../../../40-event-storming-bounded-context/BC_A_19_coupon.md), [REQ.A.02](../../../00-requirements/REQ_A_02_coupon_benefit.md)
- 결정: [Context 쿠폰 Hotspot 결정 기록](../hotspot-decisions.md)
- 책임 문서: [캠페인과 정책](campaign-policy.md), [발급](issuance.md), [사용](redemption.md), [운영과 복구](operations-recovery.md)
- 구현 설계: [원장과 신뢰성](../A_19_20-persistence/ledgers-and-reliability.md), [읽기 모델과 인덱스](../A_19_20-persistence/read-models-and-indexes.md), [이벤트 처리](../A_19_30-service/event-processing.md)

## 공통 식별자와 VO

| 계약 | 구성 | 규칙 |
| --- | --- | --- |
| `CouponIdentity` | `campaign_id`, 선택적 `user_coupon_id` | 외부 상품·주문 식별자를 쿠폰 식별자로 대체하지 않는다. |
| `BusinessKey` | `operation_type`, `owner_ref`, `source_ref` | 업무 의미가 같은 재요청은 같은 값을 사용한다. |
| `ExternalRef` | `context`, `type`, `id` | 외부 원본을 참조하며 표시명·상태 원본을 소유하지 않는다. |
| `SnapshotRef` | `source_ref`, `source_version`, `captured_at`, `payload_hash` | 판단에 사용한 외부 스냅샷을 재현할 수 있어야 한다. |
| `Money` | `amount`, `currency` | 음수를 허용하지 않고 같은 통화끼리만 계산한다. |
| `TimeWindow` | `starts_at`, `ends_at` | 시작은 종료보다 이르며 비교 기준 시각을 명시한다. |
| `Correlation` | `correlation_id`, `causation_id`, `trace_id` | Event 연쇄와 관측 추적에 사용한다. |
| `RecoveryCorrelation` | `recovery_id`, `attempt_id`, `business_key`, `result_ref` | 사용 재실행의 결과 반영 단위다. |

## 외부 경계

| 외부 Context | 입력으로 받는 것 | 내보내는 것 | 소유하지 않는 것 |
| --- | --- | --- | --- |
| 사용자·인증 | `user_id`, 차단·등급 자격 스냅샷 | 발급·만료 결과 | 계정, 인증 수단, 사용자 프로필, 생일·생년월일 |
| 판매자·상품·드롭 | 판매자 소유, 상품·드롭·카테고리 참조 스냅샷 | 허용된 성과 집계 | 판매자·상품·드롭 원본 |
| 주문·결제 | 주문 가격·구성 스냅샷, 주문·결제 결과 참조 | 할인 결과, 사용 예약·확정·해제·회수 결과 | 주문, 결제, PG 상태 |
| CS·인시던트 | 문의·승인·보상 작업 참조 | 발급·회수·실패 결과 | 문의 원문, 인시던트 원본 |
| 운영 작업 관리 | 승인된 작업 ID, 승인·사유·멱등키 | 실행 결과 Event | 승인 체계와 감사 원본 |
| 정산 | 정산 연결 요청 참조 | 비용 귀속 사실 | 정산 예정·보류·확정 상태 |
| 알림 | 없음 | 알림 판단에 필요한 도메인 Event | 채널 선택, 발송, 재시도 |

## Event 계약

모든 Domain Event는 `event_id`, `event_type`, `aggregate_type`, `aggregate_id`, `aggregate_version`, `occurred_at`, `correlation_id`, `causation_id`, `payload_schema_version`을 가진다. 소비자가 원본 Aggregate를 재구성하지 않아도 되도록 결과 식별자와 판단 스냅샷 참조를 포함하되 외부 원본 `payload` 전체를 복제하지 않는다.

외부 자동 지급 사건은 최소한 `event_id`, `user_id`, `campaign_id`, `source_ref`, `occurred_at`, schema version과 멱등키를 제공해야 한다. 생일·생년월일을 조건이나 payload로 받지 않으며, 원천 문서가 생산자·Event 유형·채널을 확정하기 전에는 계약을 배포하거나 소비자를 활성화하지 않는다.

| Event 범위 | 소유 문서 | 대표 결과 참조 |
| --- | --- | --- |
| `EVT.A.19-01`~`EVT.A.19-06`, `EVT.A.19-32`~`EVT.A.19-35` | [캠페인과 정책](campaign-policy.md) | `campaign_id`, `policy_version`, `issue_request_id` |
| `EVT.A.19-07`~`EVT.A.19-15`, `EVT.A.19-29`, `EVT.A.19-36`, `EVT.A.19-37` | [발급](issuance.md) | `issue_request_id`, `user_coupon_id`, `code_id` |
| `EVT.A.19-16`~`EVT.A.19-18` | [운영과 복구](operations-recovery.md) | `bulk_job_id` |
| `EVT.A.19-19`~`EVT.A.19-24`, `EVT.A.19-28` | [사용](redemption.md) | `redemption_id`, `order_id`, `cost_attribution_ref` |
| `EVT.A.19-25`~`EVT.A.19-27`, `EVT.A.19-30`, `EVT.A.19-31`, `EVT.A.19-38`~`EVT.A.19-41` | [운영과 복구](operations-recovery.md) | `control_id`, `recovery_id`, `attempt_id`, `result_ref` |

## Policy 계약

Policy는 Event를 입력으로 받아 다음 Command를 결정하지만 원본 Aggregate를 직접 수정하지 않는다. 같은 Event를 다시 받아도 같은 업무 고유키의 Command만 요청해야 한다.

| Policy 범위 | 소유 문서 | 연결 책임 |
| --- | --- | --- |
| `POLICY.A.19-01`~`POLICY.A.19-03`, `POLICY.A.19-07`, `POLICY.A.19-13`, `POLICY.A.19-19` | [캠페인과 정책](campaign-policy.md) | 책임 주체·승인·정책 버전·수량 |
| `POLICY.A.19-04`, `POLICY.A.19-05`, `POLICY.A.19-09`~`POLICY.A.19-11`, `POLICY.A.19-14`, `POLICY.A.19-15`, `POLICY.A.19-20` | [발급](issuance.md) | 자격·코드·발급 후속·재처리 |
| `POLICY.A.19-06` | [사용](redemption.md) | 주문 스냅샷 검증 |
| `POLICY.A.19-08` | [캠페인과 정책](campaign-policy.md), [사용](redemption.md), [운영과 복구](operations-recovery.md) | 신규 발급 수량 예약과 신규 사용 예약 차단 |
| `POLICY.A.19-12`, `POLICY.A.19-16`~`POLICY.A.19-18`, `POLICY.A.19-21`, `POLICY.A.19-22` | [운영과 복구](operations-recovery.md) | 대량 집계·자동 지급·만료·복구 |

## Business Rule 계약

| Rule | 주 책임 문서 | 구현 방어선 |
| --- | --- | --- |
| `RULE.A.19-01` | [캠페인과 정책](campaign-policy.md) | Aggregate 전이 + Postgres unique/check/lock |
| `RULE.A.19-02`, `RULE.A.19-08` | [발급](issuance.md) | 발급 요청 unique + 결과 참조 |
| `RULE.A.19-03`, `RULE.A.19-04`, `RULE.A.19-07`, `RULE.A.19-10`, `RULE.A.19-11` | [사용](redemption.md) | 사용 원장 제약 + Read Model 투영 |
| `RULE.A.19-05`, `RULE.A.19-12` | [운영과 복구](operations-recovery.md) | 복구 상관키 제약 + 상태 조건부 갱신 |
| `RULE.A.19-06` | 모든 책임 문서 | Postgres 원장·제약·outbox |
| `RULE.A.19-09` | 모든 책임 문서 | Command Handler 트랜잭션 경계 |

## Read Model 계약

| ID | 이름 | 원천 투영 | 개인정보·외부 원본 경계 |
| --- | --- | --- | --- |
| `RM.A.19-01` | 구매자 쿠폰함 | 발급 요청 + 사용자 쿠폰 + 사용 Event | `user_id` 기준 조회, 프로필 미복제 |
| `RM.A.19-02` | 쿠폰 상세·적용 조건 | 캠페인 + 사용자 쿠폰 | 외부 대상은 표시 스냅샷만 사용 |
| `RM.A.19-03` | 판매자 쿠폰 성과 | 캠페인 + 발급 + 사용 Event | 소유·부담 캠페인의 비식별 집계, `asOf`와 자기 부담 비용만 제공 |
| `RM.A.19-04` | 쿠폰 발급·사용 성과 | 대량 작업 + 발급 + 사용 Event | 집계값만 제공 |
| `RM.A.19-05` | 쿠폰 실패 이벤트 | 발급 요청 + 복구 기록 | `payload` 원문 대신 `payload_ref` 제공 |
| `RM.A.19-06` | 사용자 쿠폰 타임라인 | 발급 요청 + 사용자 쿠폰 + 사용 Event | CS 시스템의 승인된 조회 주체 참조 필요 |
| `RM.A.19-07` | 쿠폰 장애 현황 | 업무 Event + 외부 관측 지표 | Redis·MQ·Worker 상태는 보조 지표 |
| `RM.A.19-08` | 쿠폰 비용 귀속 | 사용·회수·비용 Event | 정산 상태 원본은 외부 참조 |
| `RM.A.19-09` | 쿠폰 읽기 전용 안내 | 운영 제어 Event | 안내 문구와 범위만 소유 |

## Hotspot 결정 지도

| Hotspot | 책임 문서 |
| --- | --- |
| `HOTSPOT.A.19-01` | [발급](issuance.md) |
| `HOTSPOT.A.19-02`, `HOTSPOT.A.19-03`, `HOTSPOT.A.19-08` | [사용](redemption.md) |
| `HOTSPOT.A.19-04`, `HOTSPOT.A.19-07` | [캠페인과 정책](campaign-policy.md) |
| `HOTSPOT.A.19-05` | [캠페인과 정책](campaign-policy.md), [발급](issuance.md) |
| `HOTSPOT.A.19-06`, `HOTSPOT.A.19-09` | [운영과 복구](operations-recovery.md) |
