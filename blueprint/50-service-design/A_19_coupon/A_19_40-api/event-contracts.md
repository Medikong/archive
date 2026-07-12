---
id: API.A.19.EVENTS
title: Context 쿠폰 Event 계약
type: service-design-event-contract
status: draft
tags: [service-design, coupon, event, mq, outbox, inbox]
source: local
created: 2026-07-11
updated: 2026-07-12
service_design: SD.A.19
api_design: SD.A.1940
bounded_context: BC.A.19
domain_model: SD.A.1910
service: SD.A.1930
---

# Context 쿠폰 Event 계약

## 역할

HTTP 요청 밖에서 전달되는 41개 Domain Event의 공통 envelope, outbox/inbox 전달 보장과 외부 수신 Event의 결정 경계를 정의한다. 개별 Event의 업무 의미는 BC와 책임별 도메인 문서를 원장으로 사용한다.

## 공통 envelope

| 필드 | 규칙 |
| --- | --- |
| `event_id` | 전역 고유 ID다. 재발행해도 바꾸지 않는다. |
| `event_type` | `EVT.A.19-01~41`에 대응하는 안정적인 유형이다. |
| `aggregate_type`, `aggregate_id` | Event를 만든 Aggregate를 식별한다. |
| `aggregate_version` | Aggregate에서 Event가 발생한 version이다. |
| `occurred_at` | 도메인 상태 전이가 확정된 시각이다. |
| `correlation_id`, `causation_id` | 최초 요청과 직접 원인 Event·Command를 연결한다. |
| `payload_schema_version` | payload의 호환성 version이다. 지원하지 않는 version은 실패로 기록한다. |
| `data` | 결과 식별자와 판단 SnapshotRef만 포함하고 외부 원본 payload를 복제하지 않는다. |

## 전달 보장

- Aggregate, append-only 원장과 outbox는 같은 Postgres 트랜잭션에 저장한다.
- 발행기는 같은 `event_id`로 재시도한다. 전달은 최소 한 번이며 정확히 한 번을 가정하지 않는다.
- 소비자는 `(consumer_name, event_id)` inbox를 먼저 확보하고 업무 고유키를 다시 확인한다.
- 업무 거절 Event는 정상 소비 결과다. 기술 실패, 손상된 payload, 지원하지 않는 schema만 재시도·DLQ 대상으로 남긴다.
- broker 종류, topic 이름, partition 수와 보존 기간은 현재 BC·REQ가 정하지 않았으므로 이 문서에서 새로 확정하지 않는다.

## Event 범위

| Event 범위 | 주 소비자 | payload 책임 문서 |
| --- | --- | --- |
| `EVT.A.19-01~06` | 정책 투영, 판매자·운영 조회, 발급 게이트 | [캠페인과 정책](../A_19_10-domain-model/campaign-policy.md) |
| `EVT.A.19-07~15`, `29`, `32~37` | 수량·발급·코드 Policy, 쿠폰함, 알림 | [발급](../A_19_10-domain-model/issuance.md) |
| `EVT.A.19-16~18` | 대량 발급 Worker, 성과·실패 투영 | [운영과 복구](../A_19_10-domain-model/operations-recovery.md) |
| `EVT.A.19-19~24`, `28` | 쿠폰함, 성과, 비용 귀속, 정산·알림 | [사용](../A_19_10-domain-model/redemption.md) |
| `EVT.A.19-25~27`, `30~31`, `38~41` | 운영 결과, 실패·장애 투영, 복구 Policy | [운영과 복구](../A_19_10-domain-model/operations-recovery.md) |

## 외부 수신 경계

| 생산자 | 필수 참조 | 처리 결과 | 계약 상태 |
| --- | --- | --- | --- |
| 주문·결제 | 결제 최종 확정, 확정 실패·취소, 검증된 취소·환불 ref, 주문 SnapshotRef, 업무 고유키 | 최종 확정은 `CMD.A.19-11`, 확정 전 실패·취소는 `CMD.A.19-12`, 확정 뒤 취소·환불은 `CMD.A.19-15` | 기준 사건은 확정. 구체 Event 이름과 schema는 주문·결제 원천 계약에 연결 필요 |
| 시스템 자동 지급 원천 | `event_id`, `user_id`, `campaign_id`, `source_ref`, `occurred_at`, schema version, 멱등키 | 계약 확정 뒤 `POLICY.A.19-16` → `CMD.A.19-13` | 개인정보 원칙과 필수 envelope 확정. 생산자·Event 유형·채널은 미확정 |
| 시간 스케줄러 | 만료 대상 `user_coupon_id`, 기준 시각 | `POLICY.A.19-17` → `CMD.A.19-24` | 내부 Worker 신호 |
| 운영 작업 관리 | 작업 ID, approvalRef, 사유 ref, 멱등키 | 운영·복구 Command | HTTP 계약 우선, Event는 결과 전달 보조 |

구체 Event 이름과 schema가 없는 외부 Event는 AsyncAPI나 소비자 구현 계약으로 배포하지 않는다. 주문·결제 원천 계약과 자동 지급 생산자가 확정되면 event type, schema, 호환성 정책과 채널을 함께 추가한다.

## 보안과 개인정보

- 쿠폰 코드 원문, 사용자 프로필, 생일·생년월일, 상품·주문·CS payload 원문, 승인 원문을 Event에 넣지 않는다.
- 외부 원본은 `ExternalRef` 또는 `SnapshotRef`로만 전달하고 소비 권한을 workload별로 제한한다.
- 로그와 metric label에는 `event_type`, schema version, 처리 결과와 지연만 넣고 사용자 ID·코드·payload를 넣지 않는다.

## 검증 항목

- 같은 `event_id`를 중복 수신해도 후속 Command와 Read Model 반영은 한 번이다.
- outbox 재발행 뒤에도 `event_id`, correlation과 payload version이 유지된다.
- 지원하지 않는 schema version을 성공 처리하거나 조용히 폐기하지 않는다.
- `recovery_id`, `attempt_id`, 업무 고유키와 `result_ref`가 다른 복구 결과를 반영하지 않는다.
- 외부 원본 payload와 민감 값이 Event, 로그, trace와 DLQ 운영 화면에 복제되지 않는다.
