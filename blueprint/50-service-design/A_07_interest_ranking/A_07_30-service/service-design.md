---
id: SD.A.0730
title: Context 관심/랭킹 서비스 설계
type: service-design-service
status: draft
tags: [service-design, interest, wishlist, ranking, service]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.07
bounded_context: BC.A.07
domain_model: SD.A.0710
persistence: SD.A.0720
api: SD.A.0740
---

# Context 관심/랭킹 서비스 설계

## 역할

찜 토글·조회 기록을 처리하는 Command Handler, catalog-service/order-service 이벤트를 소비하는 Event Consumer, 당일 카운터 리셋 Worker의 처리 절차와 트랜잭션 경계를 정의한다.

## 연관 태그

🏷️ BC 참조: [BC.A.07](../../../40-event-storming-bounded-context/BC_A_07_interest_ranking.md) | 도메인 참조: [SD.A.0710](../A_07_10-domain-model/SD_A_0710_interest_domain_model.md) | 영속성 참조: [SD.A.0720](../A_07_20-persistence/persistence-design.md) | API 참조: SD.A.0740 예정

## Handler 지도

2026-07-14 수정(2차): 1차 구현은 찜 토글 Handler + 찜 카운터 반영 Consumer + 조회 기록 Handler + 실시간 조회 랭킹 배치 Worker를 포함한다. "조회 기록"은 처음엔 Redis 인프라가 필요하다고 보고 후속으로 미뤘다가, Redis `SETNX` dedup 없이 원문 기록 + 집계 시점 `COUNT(DISTINCT)`로 같은 효과를 낼 수 있음을 확인해 지금 구현 범위에 포함시켰다(`REQ.A.07` 수정 이력 참고). "랭킹 리스트 전환"/"오픈 후 점수 갱신"(오픈 후 랭킹용)은 여전히 후속 스코프다. "당일 카운터 리셋 Worker"는 리셋 자체가 없어져 완전히 불필요해졌다.

| 계층 | 이름 | 트리거 | 대상 Aggregate | 정합성 | 1차 구현 |
| --- | --- | --- | --- | --- | --- |
| Command Handler | 찜 토글 Handler | HTTP Command (`CMD.A.07-01`) | `Interest` | 즉시 | 구현함 |
| Command Handler | 조회 기록 Handler | HTTP Command (`CMD.A.07-02`) | `DropView`(신규) | 지연 허용(dedup 없음, 원문 기록) | 구현함 |
| Event Consumer | 찜 카운터 반영 Consumer(구 "오픈전 카운터 반영") | 내부 이벤트: 찜 추가됨/해제됨 (`CMD.A.07-05`) | `DropInterestCounter` | 지연 허용 | 구현함 |
| Event Consumer | 랭킹 리스트 전환 Consumer | 외부 이벤트: `catalog.drop.updated` (`CMD.A.07-03`) | `DropInterestCounter` | 지연 허용 | 보류 |
| Event Consumer | 오픈 후 점수 갱신 Consumer | 외부 이벤트: `order.confirmed` (`CMD.A.07-04`) | `DropInterestCounter` | 지연 허용 | 보류 |
| Worker | ~~당일 카운터 리셋 Worker~~ | ~~스케줄(매일 00:00 KST) 또는 lazy reset~~ | ~~`DropInterestCounter`~~ | ~~지연 허용~~ | 삭제됨 — 리셋 없음 |
| Worker | 실시간 조회 랭킹 배치 Worker(신규) | 스케줄(KST 00/03/06/09/12/15/18/21시 정각) | `DropView` → `drop_view_rankings` | 지연 허용, 최대 3시간 | 구현함 |

## Command Handler 설계

### 찜 토글 Handler (`CMD.A.07-01`)

- 입력: 인증된 `user_id`, `drop_id`, 연산 방향(`PUT`=추가/`DELETE`=해제).
- 절차: 1) 로그인 게이트(`POLICY.A.07-01`) 확인 → 2) `InterestRepository.findByUserAndDrop` 조회 → 3) 목표 상태와 현재 상태가 같으면 멱등 응답(재갱신 없음, 상태전이 다이어그램의 `active→active`/`inactive→inactive`) → 4) 다르면 `upsertStatus`로 상태 전이 → 5) 커밋 → 6) 찜 추가됨/찜 해제됨 이벤트 발행.
- 트랜잭션 경계: `interests` 단일 행 갱신까지. 카운터 반영은 별도 트랜잭션(비동기 Consumer)으로 분리한다(`RULE.A.07-03`).
- 실패 처리: 낙관적 잠금(`version`) 충돌 시 재조회 후 1회 재시도, 재충돌 시 409로 응답.

### 조회 기록 Handler (`CMD.A.07-02`, 2026-07-14 재설계)

- 입력: 인증된 `user_id`, `drop_id`.
- 절차: 1) 로그인 게이트 확인(`POLICY.A.07-01`) → 2) `DropViewRepository.recordView(drop_id, user_id)`로 `drop_views`에 그대로 insert → 3) 200 응답.
- 트랜잭션 경계: `drop_views` insert 하나뿐. Redis 등 DB 트랜잭션 밖의 선행 단계가 없어져 실패 케이스가 단순해졌다(2026-07-14: 기존 "Redis 성공/DB 실패" 확인 필요 항목 해소).
- dedup은 쓰기 시점이 아니라 읽기 시점(배치 집계의 `COUNT(DISTINCT user_id)`)에서 처리한다(`POLICY.A.07-02`).

## Event Consumer 설계

### 찜 카운터 반영 Consumer(구 "오픈전 카운터 반영", `CMD.A.07-05`)

- 구독: 찜 추가됨 / 찜 해제됨(내부 이벤트, 발행처는 찜 토글 Handler).
- 절차(2026-07-14 단순화): 찜 추가는 `DropInterestCounterRepository.increment`, 찜 해제는 `decrement` 호출 — `INSERT ... ON CONFLICT DO UPDATE SET interest_count = interest_count ± 1`로 원자적 처리해 리셋 판단(`RULE.A.07-01`)이나 가중치(`weight`) 계산이 필요 없다.
- 멱등키: 확인 필요 — 같은 이벤트가 재전송되면 카운터가 중복 증감할 수 있어 이벤트 ID 기반 처리 이력이 필요하다.

### 랭킹 리스트 전환 Consumer (`CMD.A.07-03`)

- 구독: `catalog.drop.updated`(외부, `EXT.A.07-01`).
- 절차: `drop_phase` 전이(`SCHEDULED → OPEN → CLOSED`), `OPEN` 전환 시점에 `opened_at` 기록.
- 제약: `catalog.drop.updated`는 아직 `packages/contracts`에 계약으로 존재하지 않는다(`REQ.A.07` 제약 조건). 계약이 확정되기 전까지 이 Consumer는 목(mock) 이벤트로만 검증 가능하다.

### 오픈 후 점수 갱신 Consumer (`CMD.A.07-04`)

- 구독: `order.confirmed`(외부, `EXT.A.07-02`).
- 절차: `confirmed_count`/`total_quantity` 스냅샷 갱신 → `opened_at`이 설정된 경우에만 `sell_through_score = (confirmed_count / total_quantity) / elapsed_minutes`로 재계산(`RULE.A.07-02`) → `opened_at` 없으면 스킵.

## Worker 설계

### ~~당일 카운터 리셋 Worker~~ (`RULE.A.07-01`, 2026-07-14 삭제)

- 리셋 자체가 없어져 완전히 불필요해졌다. `찜 카운터 반영 Consumer`가 원자적 증감만 하면 끝이다.

### 실시간 조회 랭킹 배치 Worker (신규, 2026-07-14)

- 트리거: KST 00/03/06/09/12/15/18/21시 정각 스케줄(사용자 확정, `REQ.A.07` 수정 이력 참고).
- 절차: 방금 마감된 3시간 구간(`[bucket_start, bucket_start + 3h)`)에 대해 `DropViewRankingRepository.computeAndStoreBucket` 호출 → `drop_views`에서 `GROUP BY drop_id, COUNT(DISTINCT user_id) DESC LIMIT 100` → `drop_view_rankings`에 해당 `bucket_start`로 100행 upsert.
- **원본 정리(2026-07-14 추가)**: 스냅샷 저장에 성공한 뒤, "이번에 막 스냅샷을 만든 구간"이 아니라 **그 이전 구간**(안전 마진 1구간)의 `drop_views` 행을 삭제한다 — 즉 배치가 항상 "직전 스냅샷 구간" 원본을 지우고 "방금 만든 스냅샷 구간" 원본은 다음 배치까지 남겨둔다. 이러면 `drop_views`는 항상 최대 2개 구간(최대 6시간)분량만 유지되고, 스냅샷 계산에 문제가 있었을 때 직전 구간은 재계산할 여지가 남는다. 별도 청소 배치가 필요 없어졌다(`SD.A.0720`의 "청소 배치" 확인 필요 항목 해소).
- 대상: `drop_views`(읽기+지연 삭제), `drop_view_rankings`(쓰기).
- 실패 처리: 확인 필요(아래 참고) — 배치가 실패하면 그 구간의 스냅샷이 비게 된다.

## 트랜잭션 경계 요약

| 작업 | DB 트랜잭션 범위 | 이벤트 발행 시점 | 재시도 |
| --- | --- | --- | --- |
| 찜 토글 | `interests` 단일 행 | 커밋 후 발행(확인 필요: outbox 도입 여부) | 낙관적 잠금 충돌 시 1회 |
| 조회 기록(2026-07-14 재설계) | `drop_views` insert 하나 | 발행 없음 | 실패해도 랭킹에 소폭 영향만, 강제 재시도 없음 |
| 찜 카운터 반영 | `drop_interest_counters` 단일 행, `INSERT ... ON CONFLICT` | 없음(내부 소비만) | 멱등키 확정 후 재시도 가능 |
| 실시간 조회 랭킹 배치(신규) | `drop_view_rankings` 구간 단위 upsert(최대 100행) | 없음 | 확인 필요(재실행/백필 정책) |
| (보류) 랭킹 리스트 전환 | `drop_interest_counters` 단일 행 | `랭킹 리스트 전환됨` 발행 | 계약 확정 후 결정 |
| (보류) 오픈 후 점수 갱신 | `drop_interest_counters` 단일 행 | `오픈후 점수 갱신됨` 발행 | 계약 확정 후 결정 |

## 확인 필요

- 찜 추가/해제 이벤트를 트랜잭셔널 outbox로 발행할지, 커밋 후 직접 발행할지.
- Event Consumer들의 멱등키 설계(이벤트 ID 처리 이력 저장 위치와 보존 기간).
- `catalog.drop.updated` 계약이 `packages/contracts`에 언제 추가되는지 catalog 담당자와 협의.
- (2026-07-14 해소) ~~당일 카운터 리셋 방식~~ — 리셋 자체가 없어져 해당 없음.
- (2026-07-14 추가) 실시간 조회 랭킹 배치가 특정 구간 실행에 실패하면(서버 재시작 등) 그 구간은 스냅샷이 빈 채로 남는다. 재실행/백필 정책 미정 — 당장은 "다음 구간부터 정상 진행"으로 두고, 실제로 문제 되면 그때 정한다.
- (2026-07-14 해소) ~~`drop_views` 청소 배치~~ — 별도 배치 없이, 실시간 조회 랭킹 배치가 스냅샷 저장 후 직전 구간 원본을 지우는 방식으로 통합했다(최대 2개 구간, 6시간분만 유지).
