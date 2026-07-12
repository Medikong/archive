---
id: SD.A.1930.04
title: Context 쿠폰 이벤트 처리 설계
type: service-design-service
status: draft
tags: [service-design, coupon, event, policy, outbox, inbox, observability]
source: local
created: 2026-07-10
updated: 2026-07-12
service_design: SD.A.19
bounded_context: BC.A.19
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# Context 쿠폰 이벤트 처리 설계

## 책임

BC의 41개 Domain Event를 outbox로 전달하고, 22개 Policy와 외부 Event Handler를 inbox 기반으로 멱등 처리하는 방법을 정의한다. 외부 연동은 포트 책임까지만 다루며 HTTP·OpenAPI 계약은 정의하지 않는다.

## 연관 문서

- 원천: [BC.A.19](../../../40-event-storming-bounded-context/BC_A_19_coupon.md), [REQ.A.02](../../../00-requirements/REQ_A_02_coupon_benefit.md)
- 결정: [Context 쿠폰 Hotspot 결정 기록](../hotspot-decisions.md)
- 도메인: [공통 계약](../A_19_10-domain-model/shared-contracts.md)
- 저장: [원장과 신뢰성](../A_19_20-persistence/ledgers-and-reliability.md), [조회 모델과 인덱스](../A_19_20-persistence/read-models-and-indexes.md)
- Handler: [발급](issuance-handlers.md), [사용](redemption-handlers.md), [운영 Worker](operations-workers.md)

## 처리 원칙

- Aggregate 변경과 Domain Event outbox 기록은 같은 트랜잭션이다.
- Event Handler는 `(consumer_name, event_id)` inbox를 먼저 확보한다.
- Policy는 Event에서 다음 Command를 결정할 뿐 대상 Aggregate를 직접 저장하지 않는다.
- 후속 Command의 업무 고유키는 원인 Event와 대상 업무 의미에서 결정적으로 생성한다.
- 업무 거절 Event는 정상 소비 결과다. 기술 실패만 전달 재시도 대상으로 남긴다.
- 스키마 버전을 지원하지 못하면 조용히 건너뛰지 않고 최종 실패와 관측 신호를 남긴다.

## 내부 Policy Handler 지도

| Policy | 입력 Event | 후속 Command·판단 |
| --- | --- | --- |
| `POLICY.A.19-01` 발급 주체·비용 부담 명시 | 정책·발급 입력 | 누락 시 등록·요청 거절 |
| `POLICY.A.19-02` 판매자 소유 범위 제한 | 정책 등록 입력 | 외부 소유 스냅샷 검증 |
| `POLICY.A.19-03` 쿠폰 승인 게이트 | `EVT.A.19-02` | 승인 전 발급 차단 |
| `POLICY.A.19-04` 발급 자격·수량 제한 | 수령 입력 | `CMD.A.19-05` 허용·거절 판단 |
| `POLICY.A.19-05` 코드 등록 유효성 | 코드 입력 | `CMD.A.19-06` 허용·거절 판단 |
| `POLICY.A.19-06` 주문 스냅샷 검증 | 주문 검증 입력 | `CMD.A.19-09` 허용·거절 판단 |
| `POLICY.A.19-07` 정책 버전 적용 | `EVT.A.19-06` | 유효 버전 투영·캐시 무효화 |
| `POLICY.A.19-08` 운영 중지 우선 | `EVT.A.19-25`와 신규 업무 | 신규 수량 예약·사용 예약 거절 |
| `POLICY.A.19-09` 발급 요청 후속 처리 | `EVT.A.19-36`, `EVT.A.19-37` | `CMD.A.19-07` |
| `POLICY.A.19-10` 코드 등록 보상 처리 | `EVT.A.19-09`, `EVT.A.19-08`, `EVT.A.19-11` | `CMD.A.19-16` 또는 `CMD.A.19-17` |
| `POLICY.A.19-11` 발급 요청 생성 | `EVT.A.19-12`, `EVT.A.19-16` | `CMD.A.19-13` |
| `POLICY.A.19-12` 대량 발급 결과 반영 | `EVT.A.19-09`, `EVT.A.19-08`, `EVT.A.19-11` | `CMD.A.19-18` |
| `POLICY.A.19-13` 발급 수량 예약 | `EVT.A.19-07` | `CMD.A.19-26` |
| `POLICY.A.19-14` 발급 실패 재처리 결정 | `EVT.A.19-10` | 설정에 따라 `CMD.A.19-19` 또는 운영 확인 대기 |
| `POLICY.A.19-15` 발급 성공 요청 완료 | `EVT.A.19-09` | `CMD.A.19-23` |
| `POLICY.A.19-16` 시스템 자동 지급 요청 변환 | 외부 자동 지급 Event | `CMD.A.19-13` |
| `POLICY.A.19-17` 만료 시각 도달 처리 | 시간 스케줄 입력 | `CMD.A.19-24` |
| `POLICY.A.19-18` 만료 쿠폰 예약 해제 | `EVT.A.19-31` | 활성 예약이 있으면 `CMD.A.19-12` |
| `POLICY.A.19-19` 발급 수량 결과 반영 | `EVT.A.19-33`, `EVT.A.19-09`, `EVT.A.19-11` | `CMD.A.19-29`, `CMD.A.19-27`, `CMD.A.19-28` |
| `POLICY.A.19-20` 발급 처리 대기 전환 | `EVT.A.19-32` | `CMD.A.19-30` |
| `POLICY.A.19-21` 실패 원본 업무 재실행 | `EVT.A.19-39` | `CMD.A.19-32` |
| `POLICY.A.19-22` 재처리 결과 반영 | `EVT.A.19-41` 또는 재실행 실패 | `CMD.A.19-33` |

`POLICY.A.19-08`은 세 Aggregate의 Handler 입구에서 공통으로 조회하지만 제어 상태를 변경하지 않는다. 중지 적용 Event는 캐시를 무효화하고 각 Handler는 현재 시각의 Postgres 제어를 최종 확인한다.

## Event 소유와 소비

| Event 범위 | 주 소비자 |
| --- | --- |
| `EVT.A.19-01`~`EVT.A.19-06` | 정책 투영, 판매자·운영 조회, 발급 게이트 |
| `EVT.A.19-07`~`EVT.A.19-15` | 수량·발급·코드 Policy, 쿠폰함, 알림 |
| `EVT.A.19-16`~`EVT.A.19-18` | 대량 대상 발급, 성과·실패 투영 |
| `EVT.A.19-19`~`EVT.A.19-24`, `EVT.A.19-28` | 쿠폰함, 성과, 비용 귀속, 정산·알림 포트 |
| `EVT.A.19-25`~`EVT.A.19-27`, `EVT.A.19-30`, `EVT.A.19-38`~`EVT.A.19-41` | 운영 작업 결과, 실패·장애 투영, 복구 Policy |
| `EVT.A.19-29`, `EVT.A.19-31`~`EVT.A.19-37` | 발급 완료·만료·수량·Worker Policy와 조회 투영 |

전체 Event `payload`의 공통 필드와 각 책임 문서는 [공통 계약](../A_19_10-domain-model/shared-contracts.md#event-계약)에서 찾는다.

## 외부 수신 Event

| 생산자 | 수신 내용 | Handler 결과 |
| --- | --- | --- |
| 주문·결제 | 결제 최종 확정, 확정 실패·취소, 검증된 취소·환불 참조 | 확정은 `CMD.A.19-11`, 확정 전 실패·취소는 `CMD.A.19-12`, 확정 뒤 취소·환불은 `CMD.A.19-15` 요청 |
| 시스템 자동 지급 원천 | `event_id`, `user_id`, `campaign_id`, `source_ref`, `occurred_at`, schema version, 멱등키 | 계약이 확정된 뒤 `POLICY.A.19-16`을 거쳐 `CMD.A.19-13` 요청 |
| 시간 스케줄러 | 만료 대상 사용자 쿠폰과 기준 시각 | `POLICY.A.19-17`을 거쳐 `CMD.A.19-24` 요청 |
| 운영 작업 관리 | 승인된 중지·안내·재처리·최종 실패 작업 | 해당 운영 Command 요청 |

주문 사용 확정 기준은 결제 최종 확정 사건이다. 구체 Event 이름과 schema는 주문·결제 원천 계약에 맞춰 연결한다. 자동 지급은 생일·생년월일을 수신·저장하지 않으며, 생산자·Event 유형·채널이 원천 문서에서 확정될 때까지 소비자를 활성화하지 않는다.

## 외부 포트

| 포트 | 책임 | 실패 처리 |
| --- | --- | --- |
| `UserEligibilityPort` | 사용자·차단·등급 자격 스냅샷 조회 | 확인 불가를 자격 있음으로 간주하지 않음 |
| `SellerCatalogSnapshotPort` | 판매자 소유와 상품·드롭·카테고리 스냅샷 조회 | 버전·해시 불일치 시 정책·주문 검증 중단 |
| `OrderSnapshotPort` | 주문 가격·구성·결과 참조 조회 | 검증 가능한 스냅샷이 없으면 사용 처리 중단 |
| `OperationApprovalPort` | 운영 작업·승인 참조와 해시 확인 | 승인 원본 판단은 외부에 두고 불일치는 거절 |
| `BulkAudiencePort` | 기준 시각의 대상 식별자 페이지 조회 | page cursor와 기준 시각을 보존해 재개 |
| `SettlementEventPort` | 비용 귀속 Event 전달 | outbox 재전달; 정산 상태를 쿠폰에 쓰지 않음 |
| `NotificationEventPort` | 발급·사용·회수·만료 Event 전달 | 채널 실패는 쿠폰 업무 상태를 되돌리지 않음 |
| `ObservabilityPort` | 기술 지표 참조와 trace 연결 | 업무 원장의 대체 저장소로 사용하지 않음 |

## 재시도와 DLQ

- outbox 발행 실패는 같은 `event_id`로 재시도한다.
- inbox 처리 실패는 같은 소비자·Event 키로 재시도한다.
- 최대 시도 뒤 DLQ로 이동해도 자동으로 `failed_final`로 바꾸지 않고 `CouponIssueRequest` 또는 `CouponEventRecovery`의 운영 확인 대기 상태를 별도로 기록한다.
- 손상된 `payload`, 지원하지 않는 스키마, 상관키 불일치는 자동 재시도를 멈추고 운영 확인 대상으로 남긴다.
- 재처리 시 새 Domain Event를 위조하지 않고 원본 Event 또는 승인된 복구 Command를 사용한다.

## 관측

| 신호 | 핵심 속성 |
| --- | --- |
| Command 처리 | `command_id`, `aggregate_type/id`, `business_key`, 결과, 지연 시간 |
| Event 전달 | `event_id/type`, outbox 지연, publish attempt, broker 참조 |
| Event 소비 | consumer, inbox 상태, 처리 지연, 후속 Command 참조 |
| 발급 정합성 | 요청·수량 예약·처리 대기·실제 발급·최종 실패 수 |
| 사용 정합성 | 검증·예약·확정·해제·회수 수와 활성 예약 수 |
| 복구 | `recovery_id`, `attempt_id`, 업무 고유키, `result_kind`, 다음 처리 시각 |
| 기술 계층 | Redis 지연·오류, MQ lag·DLQ, Worker 처리량·실패율 |

로그에는 코드 원문, 외부 `payload` 원문, 사용자 프로필을 남기지 않는다. `trace_id`, 내부 식별자와 승인된 외부 참조만 기록한다.

## Hotspot 결정 반영

| Hotspot | 반영 내용 |
| --- | --- |
| `HOTSPOT.A.19-01` | 발급 대기와 완료 Event 투영을 구분한다. |
| `HOTSPOT.A.19-02`, `HOTSPOT.A.19-03` | 결제 최종 확정과 즉시 해제·제한적 재사용을 연결한다. |
| `HOTSPOT.A.19-05`, `HOTSPOT.A.19-06` | 위험 기반 승인, 설정 기반 백오프와 승인된 최종 실패를 적용한다. |
| `HOTSPOT.A.19-07`, `HOTSPOT.A.19-08` | 판매자 비식별 집계와 명시적 중복 적용 정책을 투영한다. |
| `HOTSPOT.A.19-09` | 개인정보 원칙과 필수 envelope는 확정했다. 생산자·Event 유형·채널은 남은 계약 결정이다. |
