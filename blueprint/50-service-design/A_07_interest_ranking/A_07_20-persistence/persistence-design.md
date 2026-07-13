---
id: SD.A.0720
title: Context 관심/랭킹 영속성 설계
type: service-design-persistence
status: draft
tags: [service-design, interest, wishlist, ranking, persistence]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.07
bounded_context: BC.A.07
domain_model: SD.A.0710
service: SD.A.0730
api: SD.A.0740
---

# Context 관심/랭킹 영속성 설계

## 역할

`Interest`(즉시 정합성)와 `DropInterestCounter`(지연 허용)를 저장하는 방식과 저장소 분리 근거, Repository 경계, 조회수 dedup의 인프라 처리를 정의한다.

## 연관 태그

🏷️ BC 참조: [BC.A.07](../../../40-event-storming-bounded-context/BC_A_07_interest_ranking.md) | 도메인 참조: [SD.A.0710](../A_07_10-domain-model/SD_A_0710_interest_domain_model.md) | 서비스 참조: SD.A.0730 예정 | API 참조: SD.A.0740 예정

## 저장 원칙

- `Interest`는 사용자에게 즉시 정확해야 하므로 관계형 DB(Postgres)에 단일 트랜잭션으로 쓴다(`RULE.A.07-03`).
- `DropInterestCounter`는 지연을 허용하므로 1차 구현은 Postgres로 시작하고, 핫키 실측(`HOTSPOT.A.07-01`) 결과에 따라 Redis `ZINCRBY` 기반 저장으로 전환할지 결정한다. 지금 단계에서 Redis를 기본값으로 확정하지 않는다 — 운영 복잡도를 트래픽이 실제로 요구할 때만 늘린다.
- 조회수 dedup(`POLICY.A.07-02`)은 도메인 Aggregate가 아니라 인프라 idempotency 가드로 취급한다. Redis 키 `view:{user_id}:{drop_id}`를 `SETNX` + `EX 300`(5분)으로 사용해 동일 조합의 중복 조회를 걸러내고, 통과한 요청만 `DropInterestCounter`에 반영한다.
- 삭제 대신 상태값(`status`, `drop_phase`)으로 종단 상태를 표현한다. 찜 해제는 레코드 삭제가 아니라 `status=inactive` 전이다(`AGG.A.07-01` 상태 전이 참고).

## 저장 모델

| 테이블/컬렉션 | 역할 | 연결 도메인 | 비고 |
| --- | --- | --- | --- |
| `interests` (Postgres) | 사용자-드롭 찜 상태, 즉시 정합성 | `Interest` (`AGG.A.07-01`) | `(user_id, drop_id)` unique |
| `drop_interest_counters` (Postgres, 1차) | 드롭별 오픈전 누적/오픈후 점수/조회수, 지연 허용 | `DropInterestCounter` (`AGG.A.07-02`) | 핫키 실측 후 Redis 전환 검토 |
| `view:{user_id}:{drop_id}` (Redis, TTL) | 조회 dedup 가드 | 도메인 Aggregate 아님, 인프라 idempotency | `SETNX` + `EX 300` |

## 스키마

| 저장 모델 | 필드 | 타입 | 제약 | 설명 |
| --- | --- | --- | --- | --- |
| `interests` | `interest_id` | uuid | PK | 찜 레코드 식별자 |
| `interests` | `user_id` | uuid | not null | 찜한 사용자 |
| `interests` | `drop_id` | uuid | not null | 찜 대상 드롭 |
| `interests` | `status` | enum(`active`,`inactive`) | not null | 찜 토글 상태 |
| `interests` | `created_at` | timestamptz | not null | 최초 생성 시각 |
| `interests` | `updated_at` | timestamptz | not null | 최근 토글 시각 |
| `interests` | `version` | bigint | not null, default 0 | 낙관적 잠금 |
| `drop_interest_counters` | `drop_id` | uuid | PK | 드롭 식별자 |
| `drop_interest_counters` | `drop_phase` | enum(`SCHEDULED`,`OPEN`,`CLOSED`) | not null | catalog-service 스냅샷 |
| `drop_interest_counters` | `upcoming_count` | int | not null, default 0, >= 0 | 오픈 전 당일 누적 카운트 |
| `drop_interest_counters` | `upcoming_count_date` | date | not null | 리셋 기준일 |
| `drop_interest_counters` | `view_count_total` | int | not null, default 0 | 누적 조회수 |
| `drop_interest_counters` | `total_quantity` | int | null | order-service 스냅샷 |
| `drop_interest_counters` | `confirmed_count` | int | null | order-service 스냅샷 |
| `drop_interest_counters` | `sell_through_score` | float | null | 오픈 후 점수 |
| `drop_interest_counters` | `opened_at` | timestamptz | null | 오픈 시각 |
| `drop_interest_counters` | `updated_at` | timestamptz | not null | 최근 갱신 시각 |
| `drop_interest_counters` | `version` | bigint | not null, default 0 | 낙관적 잠금(핫키 write 경합) |

## Aggregate 매핑

| 도메인 모델 | 저장 모델 | 매핑 방식 | 주의점 |
| --- | --- | --- | --- |
| `Interest` (`AGG.A.07-01`) | `interests` | 1:1, 하위 테이블 없음 | 토글은 새 행이 아니라 같은 행의 `status` 갱신 |
| `DropInterestCounter` (`AGG.A.07-02`) | `drop_interest_counters` | 1:1, 하위 테이블 없음 | `total_quantity`/`confirmed_count`는 order-service의 참조 스냅샷이며 이 BC가 원본을 소유하지 않음 |

## Repository 설계 근거

| Repository 메서드 | 저장 모델 | 쿼리 기준 | 반환/저장 대상 |
| --- | --- | --- | --- |
| `InterestRepository.findByUserAndDrop` | `interests` | `(user_id, drop_id)` | 토글 전 현재 상태 조회 |
| `InterestRepository.upsertStatus` | `interests` | `WHERE interest_id = ? AND version = ?` | 상태 전이, 낙관적 잠금 |
| `InterestRepository.listActiveByUser` | `interests` | `WHERE user_id = ? AND status = 'active'` | 찜 목록 조회(`RM.A.07-01`) |
| `DropInterestCounterRepository.findByDrop` | `drop_interest_counters` | `drop_id` | 카운터 단건 조회 |
| `DropInterestCounterRepository.applyUpcomingDelta` | `drop_interest_counters` | `WHERE drop_id = ? AND upcoming_count_date = ?` | 찜/조회 반영에 따른 증감(대칭, `RULE.A.07-05`) |
| `DropInterestCounterRepository.applyView` | `drop_interest_counters` | `WHERE drop_id = ?` | dedup 통과 조회수 반영 |
| `DropInterestCounterRepository.transitionPhase` | `drop_interest_counters` | `WHERE drop_id = ? AND version = ?` | `catalog.drop.updated` 반영 |
| `DropInterestCounterRepository.updateSellThroughScore` | `drop_interest_counters` | `WHERE drop_id = ? AND version = ?` | 오픈 후 점수 갱신 |
| `DropInterestCounterRepository.listByPhaseOrderByScore` | `drop_interest_counters` | `WHERE drop_phase = 'OPEN' ORDER BY sell_through_score DESC` | 오픈 후 랭킹(`RM.A.07-02`) |
| `DropInterestCounterRepository.listByPhaseOrderByUpcoming` | `drop_interest_counters` | `WHERE drop_phase = 'SCHEDULED' ORDER BY upcoming_count DESC` | 오픈 예정 랭킹(`RM.A.07-03`) |

## 쓰기 전략

| 작업 | 트랜잭션 경계 | 정합성 기준 | 실패 처리 |
| --- | --- | --- | --- |
| 찜 추가/해제 | `interests` 단일 행 | 즉시 정합성 | 낙관적 잠금 충돌 시 재조회 후 재시도 |
| 오픈전 카운터 반영(찜 이벤트 소비) | `drop_interest_counters` 단일 행 | 지연 허용(비동기) | 이벤트 재처리 멱등키 설계 필요(확인 필요) |
| 조회 기록 | Redis dedup 통과 후 `drop_interest_counters` 갱신 | 지연 허용 | dedup 실패는 오류가 아니라 무시 |
| 랭킹 리스트 전환 / 오픈후 점수 갱신 | `drop_interest_counters` 단일 행 | 지연 허용 | 외부 이벤트 재전송 시 멱등 처리 필요 |

## 읽기 전략

| 화면/API | 조회 모델 | 인덱스 | 캐시 여부 |
| --- | --- | --- | --- |
| 찜 목록 조회 | `interests` | `(user_id, status)` | 캐시 없음(즉시 정합성 요구) |
| 오픈 후 인기 랭킹 조회 | `drop_interest_counters` | `(drop_phase, sell_through_score DESC)` | 짧은 TTL 캐시 검토(확인 필요) |
| 오픈 예정 랭킹 조회 | `drop_interest_counters` | `(drop_phase, upcoming_count DESC)` | 짧은 TTL 캐시 검토(확인 필요) |
| 드롭 관심도 통계 조회 | `drop_interest_counters` | PK(`drop_id`) | 캐시 없음 |

## 인덱스

| 저장 모델 | 인덱스 | 목적 |
| --- | --- | --- |
| `interests` | unique `(user_id, drop_id)` | 찜 레코드 유일성 보장 |
| `interests` | `(user_id, status)` | 찜 목록 조회 |
| `drop_interest_counters` | `(drop_phase, sell_through_score DESC)` | 오픈 후 랭킹 정렬 조회 |
| `drop_interest_counters` | `(drop_phase, upcoming_count DESC)` | 오픈 예정 랭킹 정렬 조회 |

## 마이그레이션

- `interests`, `drop_interest_counters` 모두 신규 테이블이다. 기존 catalog-service/order-service 스키마와 무관하며 수정하지 않는다(`REQ.A.07.NFR-002`).
- 초기 마이그레이션에서 두 테이블과 위 인덱스를 함께 생성한다. 별도 백필 데이터는 없다(신규 서비스 시작).

## 확인 필요

- `upcoming_count_date` 리셋을 배치 작업으로 할지, 조회/갱신 시점의 lazy reset으로 할지.
- `DropInterestCounter` 이벤트 소비(찜 반영, 드롭 상태 전환, 점수 갱신)의 멱등키 설계 — 같은 이벤트 재전송 시 중복 반영 방지.
- 오픈 후/오픈 예정 랭킹 조회에 캐시를 둘지, 둔다면 TTL을 얼마로 할지.
- `DropInterestCounter`를 Postgres에서 Redis로 옮기는 기준선(`HOTSPOT.A.07-01` 실측 지표)을 언제 정할지.
