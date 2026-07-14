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
- `DropInterestCounter`는 지연을 허용하므로 1차 구현은 Postgres로 시작하고, 핫키 실측(`HOTSPOT.A.07-01`) 결과에 따라 Redis 기반 저장으로 전환할지 결정한다. 지금 단계에서 Redis를 기본값으로 확정하지 않는다 — 운영 복잡도를 트래픽이 실제로 요구할 때만 늘린다.
- (2026-07-14 재설계) 조회수 dedup(`POLICY.A.07-02`)은 더는 쓰기 시점 Redis 가드가 아니다. 조회를 막지 않고 `drop_views`에 원문 그대로 insert하고, "실시간 많이 보고 있는 상품" 랭킹은 KST 3시간 고정 구간(00/03/06/09/12/15/18/21시)마다 배치가 직전 구간의 `COUNT(DISTINCT user_id)`를 계산해 `drop_view_rankings` 스냅샷에 저장한다. API는 스냅샷만 읽어 매 요청마다 무거운 집계 쿼리가 돌지 않는다. 남은 위험은 조회 기록 쓰기 경로의 핫키뿐이며, 이는 `HOTSPOT.A.07-01`(찜 카운터와 동일 성격)로 실측 후 결정한다.
- `drop_views`는 무한정 쌓이지 않는다(2026-07-14 해소) — 실시간 조회 랭킹 배치 Worker가 3시간마다 스냅샷을 저장한 직후, 직전 구간의 원본 행을 지운다. 항상 최대 2개 구간(6시간)분만 유지된다. 별도 청소 배치는 없다.
- 삭제 대신 상태값(`status`, `drop_phase`)으로 종단 상태를 표현한다. 찜 해제는 레코드 삭제가 아니라 `status=inactive` 전이다(`AGG.A.07-01` 상태 전이 참고).

## 저장 모델

2026-07-14 수정: `drop_interest_counters`는 1차 구현 스코프(누적 활성 찜 수만)로 좁혔다. `drop_phase`/order-service 스냅샷 컬럼은 후속 스코프이며, 아래 스키마 표에서 "1차 구현" 열로 구분한다. 조회수는 Redis dedup 대신 `drop_views`(원문 기록) + `drop_view_rankings`(3시간 배치 스냅샷) 조합으로 1차 구현에 포함한다.

| 테이블/컬렉션 | 역할 | 연결 도메인 | 비고 |
| --- | --- | --- | --- |
| `interests` (Postgres) | 사용자-드롭 찜 상태, 즉시 정합성 | `Interest` (`AGG.A.07-01`) | `(user_id, drop_id)` unique |
| `drop_interest_counters` (Postgres, 1차) | 드롭별 누적 활성 찜 수(1차), 오픈후 점수(후속) | `DropInterestCounter` (`AGG.A.07-02`) | 핫키 실측 후 Redis 전환 검토 |
| `drop_views` (Postgres, 신규) | 조회 원문 기록(insert-only, dedup 없음) | `DropView` (신규, 아래 도메인 모델 참고) | 실시간 조회 랭킹 배치가 스냅샷 저장 후 직전 구간을 지워 최대 6시간분만 유지 |
| `drop_view_rankings` (Postgres, 신규) | KST 3시간 고정 구간별 상위 랭킹 스냅샷(Top 100) | `DropView` | 3시간마다 배치가 갱신, API는 이것만 읽음 |

## 스키마

| 저장 모델 | 필드 | 타입 | 제약 | 설명 | 1차 구현(2026-07-14) |
| --- | --- | --- | --- | --- | --- |
| `interests` | `interest_id` | uuid | PK | 찜 레코드 식별자 | 구현함 |
| `interests` | `user_id` | uuid | not null | 찜한 사용자 | 구현함 |
| `interests` | `drop_id` | uuid | not null | 찜 대상 드롭 | 구현함 |
| `interests` | `status` | enum(`active`,`inactive`) | not null | 찜 토글 상태 | 구현함 |
| `interests` | `created_at` | timestamptz | not null | 최초 생성 시각 | 구현함 |
| `interests` | `updated_at` | timestamptz | not null | 최근 토글 시각 | 구현함 |
| `interests` | `version` | bigint | not null, default 0 | 낙관적 잠금 | 구현함 |
| `drop_interest_counters` | `drop_id` | uuid | PK | 드롭 식별자 | 구현함 |
| `drop_interest_counters` | `drop_phase` | enum(`SCHEDULED`,`OPEN`,`CLOSED`) | not null | catalog-service 스냅샷 | 보류 |
| `drop_interest_counters` | `interest_count`(구 `upcoming_count`) | int | not null, default 0, >= 0 | 리셋 없는 누적 활성 찜 수 | 구현함 |
| ~~`drop_interest_counters`~~ | ~~`upcoming_count_date`~~ | ~~date~~ | ~~리셋 기준일~~ | 삭제됨 — 리셋 없음 | - |
| ~~`drop_interest_counters`~~ | ~~`view_count_total`~~ | ~~int~~ | ~~누적 조회수~~ | 대체됨 — `drop_views`/`drop_view_rankings` 조합으로 구현(2026-07-14) | - |
| `drop_interest_counters` | `total_quantity` | int | null | order-service 스냅샷 | 보류 |
| `drop_interest_counters` | `confirmed_count` | int | null | order-service 스냅샷 | 보류 |
| `drop_interest_counters` | `sell_through_score` | float | null | 오픈 후 점수 | 보류 |
| `drop_interest_counters` | `opened_at` | timestamptz | null | 오픈 시각 | 보류 |
| `drop_interest_counters` | `updated_at` | timestamptz | not null | 최근 갱신 시각 | 구현함 |
| ~~`drop_interest_counters`~~ | ~~`version`~~ | ~~bigint~~ | ~~낙관적 잠금~~ | 미사용 — `INSERT ... ON CONFLICT DO UPDATE SET interest_count = interest_count + delta`로 원자적 증감 처리해 낙관적 잠금이 불필요해짐(2026-07-14) | - |
| `drop_views`(신규, 2026-07-14) | `id` | bigserial | PK | 조회 기록 식별자 | 구현함 |
| `drop_views` | `drop_id` | uuid | not null | 조회 대상 드롭 | 구현함 |
| `drop_views` | `user_id` | uuid | not null | 조회한 사용자(로그인 사용자만, `POLICY.A.07-01`) | 구현함 |
| `drop_views` | `viewed_at` | timestamptz | not null | 조회 시각 | 구현함 |
| `drop_view_rankings`(신규, 2026-07-14) | `bucket_start` | timestamptz | PK(복합) | KST 3시간 구간 시작 시각(00/03/06/09/12/15/18/21시) | 구현함 |
| `drop_view_rankings` | `rank` | int | PK(복합), 1~100 | Top 100 순위(홈 위젯은 상위 3개만, 전체보기는 최대 100개 요청 — `limit` 파라미터로 조절) | 구현함 |
| `drop_view_rankings` | `drop_id` | uuid | not null | 해당 순위 드롭 | 구현함 |
| `drop_view_rankings` | `viewer_count` | int | not null | 해당 구간의 서로 다른 조회자 수 | 구현함 |
| `drop_view_rankings` | `computed_at` | timestamptz | not null | 배치 실행 시각 | 구현함 |

## Aggregate 매핑

| 도메인 모델 | 저장 모델 | 매핑 방식 | 주의점 |
| --- | --- | --- | --- |
| `Interest` (`AGG.A.07-01`) | `interests` | 1:1, 하위 테이블 없음 | 토글은 새 행이 아니라 같은 행의 `status` 갱신 |
| `DropInterestCounter` (`AGG.A.07-02`) | `drop_interest_counters` | 1:1, 하위 테이블 없음 | `total_quantity`/`confirmed_count`는 order-service의 참조 스냅샷이며 이 BC가 원본을 소유하지 않음 |
| `DropView`(신규, 2026-07-14) | `drop_views` | append-only, 하위 테이블 없음 | 삭제 없음. 랭킹은 이 테이블에서 파생되는 별도 스냅샷(`drop_view_rankings`)에서 읽는다 |

## Repository 설계 근거

2026-07-14 수정: `DropInterestCounterRepository`의 메서드를 실제 구현 범위(누적 증감만)에 맞게 다시 정했다. `applyUpcomingDelta`(리셋 판단 포함)는 `increment`/`decrement`(원자적 증감, 리셋 없음)로 나눠 단순화했고, `listByPhaseOrderByUpcoming`은 phase 필터가 없어져 `listByInterestCount`로 이름을 바꿨다. `applyView`/`transitionPhase`/`updateSellThroughScore`/`listByPhaseOrderByScore`는 후속 스코프로 남긴다(목표 설계 그대로 유지).

| Repository 메서드 | 저장 모델 | 쿼리 기준 | 반환/저장 대상 | 1차 구현(2026-07-14) |
| --- | --- | --- | --- | --- |
| `InterestRepository.findByUserAndDrop` | `interests` | `(user_id, drop_id)` | 토글 전 현재 상태 조회 | 구현함 |
| `InterestRepository.upsertStatus` | `interests` | `WHERE interest_id = ? AND version = ?` | 상태 전이, 낙관적 잠금 | 구현함 |
| `InterestRepository.listActiveByUser` | `interests` | `WHERE user_id = ? AND status = 'active'` | 찜 목록 조회(`RM.A.07-01`) | 구현함 |
| `DropInterestCounterRepository.get` | `drop_interest_counters` | `drop_id` | 카운터 단건 조회 | 구현함 |
| `DropInterestCounterRepository.increment`/`decrement`(구 `applyUpcomingDelta`) | `drop_interest_counters` | `INSERT ... ON CONFLICT(drop_id) DO UPDATE SET interest_count = interest_count ± 1` | 찜 추가/해제 반영에 따른 원자적 증감(대칭, `RULE.A.07-05`), 리셋 없음 | 구현함 |
| `DropInterestCounterRepository.applyView` | `drop_interest_counters` | `WHERE drop_id = ?` | dedup 통과 조회수 반영 | 보류(Redis 필요) |
| `DropInterestCounterRepository.transitionPhase` | `drop_interest_counters` | `WHERE drop_id = ? AND version = ?` | `catalog.drop.updated` 반영 | 보류 |
| `DropInterestCounterRepository.updateSellThroughScore` | `drop_interest_counters` | `WHERE drop_id = ? AND version = ?` | 오픈 후 점수 갱신 | 보류 |
| `DropInterestCounterRepository.listByPhaseOrderByScore` | `drop_interest_counters` | `WHERE drop_phase = 'OPEN' ORDER BY sell_through_score DESC` | 오픈 후 랭킹(`RM.A.07-02`) | 보류 |
| `DropInterestCounterRepository.listByInterestCount`(구 `listByPhaseOrderByUpcoming`) | `drop_interest_counters` | `ORDER BY interest_count DESC, drop_id ASC` (phase 필터 없음) | 기다리는 상품 랭킹(`RM.A.07-03`) | 구현함 |
| `DropViewRepository.recordView`(신규) | `drop_views` | `INSERT` | 조회 원문 기록(dedup 없음) | 구현함 |
| `DropViewRankingRepository.computeAndStoreBucket`(신규, 배치 전용) | `drop_views` 읽음 → `drop_view_rankings` 씀 | `SELECT drop_id, COUNT(DISTINCT user_id) FROM drop_views WHERE viewed_at >= ? AND viewed_at < ? GROUP BY drop_id ORDER BY 2 DESC LIMIT 100` | KST 3시간 구간 마감 시 Top 100 스냅샷 계산·저장 | 구현함 |
| `DropViewRankingRepository.getLatestBucket`(신규) | `drop_view_rankings` | `WHERE bucket_start = (SELECT MAX(bucket_start) ...)` | 실시간 많이 보는 상품 랭킹(`RM.A.07-06`) | 구현함 |

## 쓰기 전략

| 작업 | 트랜잭션 경계 | 정합성 기준 | 실패 처리 |
| --- | --- | --- | --- |
| 찜 추가/해제 | `interests` 단일 행 | 즉시 정합성 | 낙관적 잠금 충돌 시 재조회 후 재시도 |
| 오픈전 카운터 반영(찜 이벤트 소비) | `drop_interest_counters` 단일 행, `INSERT ... ON CONFLICT` 원자적 증감(2026-07-14 수정: 낙관적 잠금 재조회 불필요) | 지연 허용(비동기) | 이벤트 재처리 멱등키 설계 필요(미해소, 아래 확인 필요 참고) |
| 조회 기록(2026-07-14 구현 전환) | `drop_views` insert(dedup 없음) | 지연 허용 | 실패해도 랭킹에 소폭 영향만 있고 재시도 강제 안 함 |
| 실시간 조회 랭킹 배치(2026-07-14 신규) | `drop_view_rankings` bucket 단위 upsert | 지연 허용, KST 3시간 경계마다 1회 실행 | 배치 실패 시 다음 구간 스냅샷은 정상 진행, 놓친 구간은 결측으로 남김(확인 필요: 재실행/백필 정책) |
| (보류) 랭킹 리스트 전환 / 오픈후 점수 갱신 | `drop_interest_counters` 단일 행 | 지연 허용 | 외부 이벤트 재전송 시 멱등 처리 필요 |

## 읽기 전략

| 화면/API | 조회 모델 | 인덱스 | 캐시 여부 |
| --- | --- | --- | --- |
| 찜 목록 조회 | `interests` | `(user_id, status)` | 캐시 없음(즉시 정합성 요구) |
| (보류) 오픈 후 인기 랭킹 조회 | `drop_interest_counters` | `(drop_phase, sell_through_score DESC)` | 짧은 TTL 캐시 검토(확인 필요) |
| 기다리는 상품 랭킹 조회(구 "오픈 예정 랭킹", 2026-07-14 개명) | `drop_interest_counters` | `(interest_count DESC, drop_id)` | 짧은 TTL 캐시 검토(확인 필요) |
| 드롭 관심도 통계 조회 | `drop_interest_counters` | PK(`drop_id`) | 캐시 없음 |
| 실시간 많이 보는 상품 랭킹 조회(2026-07-14 신규) | `drop_view_rankings` | `bucket_start DESC` | 캐시 불필요 — 이미 3시간 배치 스냅샷이라 그 자체가 캐시 역할 |

## 인덱스

| 저장 모델 | 인덱스 | 목적 |
| --- | --- | --- |
| `interests` | unique `(user_id, drop_id)` | 찜 레코드 유일성 보장 |
| `interests` | `(user_id, status)` | 찜 목록 조회 |
| (보류) `drop_interest_counters` | `(drop_phase, sell_through_score DESC)` | 오픈 후 랭킹 정렬 조회 |
| `drop_interest_counters` | `(interest_count DESC, drop_id)` | 기다리는 상품 랭킹 정렬 조회(2026-07-14: `drop_phase` 필터 제거) |
| `drop_views` | `(drop_id, viewed_at)` | 3시간 배치 집계 쿼리(`GROUP BY drop_id`, 구간 필터) |
| `drop_view_rankings` | PK `(bucket_start, rank)` | 최신 구간 스냅샷 조회 |

## 마이그레이션

- `interests`, `drop_interest_counters`, `drop_views`, `drop_view_rankings` 모두 신규 테이블이다. 기존 catalog-service/order-service 스키마와 무관하며 수정하지 않는다(`REQ.A.07.NFR-002`).
- 초기 마이그레이션에서 네 테이블과 위 인덱스를 함께 생성한다. 별도 백필 데이터는 없다(신규 서비스 시작).

## 확인 필요

- (2026-07-14 해소) ~~`upcoming_count_date` 리셋 방식~~ — 리셋 자체가 없어져 해당 없음.
- `DropInterestCounter` 이벤트 소비(찜 반영)의 멱등키 설계 — 같은 이벤트 재전송 시 중복 반영 방지. 원자적 증감(`INSERT ... ON CONFLICT`)으로 바뀌었어도 이 문제는 그대로 남는다(2026-07-14 확인: 낙관적 잠금 제거가 멱등성 문제를 해결하진 않는다).
- 기다리는 상품 랭킹 조회에 캐시를 둘지, 둔다면 TTL을 얼마로 할지.
- `DropInterestCounter`를 Postgres에서 Redis로 옮기는 기준선(`HOTSPOT.A.07-01` 실측 지표)을 언제 정할지.
- (2026-07-14 해소) ~~"실시간 많이 보고 있는 상품" 착수 전 시간창/저장방식 결정~~ — KST 3시간 고정 구간 + Postgres(`drop_views`/`drop_view_rankings`)로 확정, 이번 슬라이스에 포함.
- (2026-07-14 해소) ~~`drop_views` 청소 배치~~ — 실시간 조회 랭킹 배치에 지연 삭제(직전 구간 원본 삭제)를 통합해 별도 배치 없이 최대 6시간분만 유지하도록 확정했다.
- (2026-07-14 추가) 조회 기록 쓰기 경로의 핫키 대응(로컬 버퍼링 등)은 `HOTSPOT.A.07-01`과 함께 부하테스트 이후 결정.
- (2026-07-14 추가) 3시간 배치가 특정 구간에 실패하면 그 구간은 스냅샷이 비는데, 재실행/백필 정책이 미정.
