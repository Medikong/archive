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

| 계층 | 이름 | 트리거 | 대상 Aggregate | 정합성 |
| --- | --- | --- | --- | --- |
| Command Handler | 찜 토글 Handler | HTTP Command (`CMD.A.07-01`) | `Interest` | 즉시 |
| Command Handler | 조회 기록 Handler | HTTP Command (`CMD.A.07-02`) | `DropInterestCounter` | 지연 허용(dedup 통과 후) |
| Event Consumer | 오픈전 카운터 반영 Consumer | 내부 이벤트: 찜 추가됨/해제됨 (`CMD.A.07-05`) | `DropInterestCounter` | 지연 허용 |
| Event Consumer | 랭킹 리스트 전환 Consumer | 외부 이벤트: `catalog.drop.updated` (`CMD.A.07-03`) | `DropInterestCounter` | 지연 허용 |
| Event Consumer | 오픈 후 점수 갱신 Consumer | 외부 이벤트: `order.confirmed` (`CMD.A.07-04`) | `DropInterestCounter` | 지연 허용 |
| Worker | 당일 카운터 리셋 Worker | 스케줄(매일 00:00 KST) 또는 lazy reset | `DropInterestCounter` | 지연 허용 |

## Command Handler 설계

### 찜 토글 Handler (`CMD.A.07-01`)

- 입력: 인증된 `user_id`, `drop_id`, 연산 방향(`PUT`=추가/`DELETE`=해제).
- 절차: 1) 로그인 게이트(`POLICY.A.07-01`) 확인 → 2) `InterestRepository.findByUserAndDrop` 조회 → 3) 목표 상태와 현재 상태가 같으면 멱등 응답(재갱신 없음, 상태전이 다이어그램의 `active→active`/`inactive→inactive`) → 4) 다르면 `upsertStatus`로 상태 전이 → 5) 커밋 → 6) 찜 추가됨/찜 해제됨 이벤트 발행.
- 트랜잭션 경계: `interests` 단일 행 갱신까지. 카운터 반영은 별도 트랜잭션(비동기 Consumer)으로 분리한다(`RULE.A.07-03`).
- 실패 처리: 낙관적 잠금(`version`) 충돌 시 재조회 후 1회 재시도, 재충돌 시 409로 응답.

### 조회 기록 Handler (`CMD.A.07-02`)

- 입력: 인증된 `user_id`, `drop_id`.
- 절차: 1) 로그인 게이트 확인 → 2) Redis `SETNX view:{user_id}:{drop_id}` + `EX 300` → 3) 키가 이미 있으면(dedup 실패) 카운터를 건드리지 않고 200으로 응답(`POLICY.A.07-02`) → 4) 키 설정에 성공하면 `DropInterestCounterRepository.applyView` 호출.
- 트랜잭션 경계: `drop_interest_counters` 단일 행. Redis dedup 키 설정은 DB 트랜잭션 밖의 선행 단계다 — Redis는 성공했는데 DB 갱신이 실패하는 경우의 처리는 확인 필요.

## Event Consumer 설계

### 오픈전 카운터 반영 Consumer (`CMD.A.07-05`)

- 구독: 찜 추가됨 / 찜 해제됨(내부 이벤트, 발행처는 찜 토글 Handler).
- 절차: `upcoming_count_date`가 오늘이 아니면 먼저 0으로 리셋(`RULE.A.07-01`) → 찜 추가는 `+weight`, 찜 해제는 `-weight`로 대칭 반영(`RULE.A.07-05`), 0 밑으로 내려가지 않게 방어.
- 멱등키: 확인 필요 — 같은 이벤트가 재전송되면 카운터가 중복 증감할 수 있어 이벤트 ID 기반 처리 이력이 필요하다.

### 랭킹 리스트 전환 Consumer (`CMD.A.07-03`)

- 구독: `catalog.drop.updated`(외부, `EXT.A.07-01`).
- 절차: `drop_phase` 전이(`SCHEDULED → OPEN → CLOSED`), `OPEN` 전환 시점에 `opened_at` 기록.
- 제약: `catalog.drop.updated`는 아직 `packages/contracts`에 계약으로 존재하지 않는다(`REQ.A.07` 제약 조건). 계약이 확정되기 전까지 이 Consumer는 목(mock) 이벤트로만 검증 가능하다.

### 오픈 후 점수 갱신 Consumer (`CMD.A.07-04`)

- 구독: `order.confirmed`(외부, `EXT.A.07-02`).
- 절차: `confirmed_count`/`total_quantity` 스냅샷 갱신 → `opened_at`이 설정된 경우에만 `sell_through_score = (confirmed_count / total_quantity) / elapsed_minutes`로 재계산(`RULE.A.07-02`) → `opened_at` 없으면 스킵.

## Worker 설계

### 당일 카운터 리셋 Worker (`RULE.A.07-01`)

- 트리거 후보: 매일 00:00 KST 배치, 또는 조회/갱신 시점에 `upcoming_count_date`를 비교해 초기화하는 lazy reset.
- 대상: `drop_interest_counters`. 배치 방식이면 전체 스캔, lazy 방식이면 해당 요청이 건드리는 행만.
- 최종 방식은 [SD.A.0720의 확인 필요](../A_07_20-persistence/persistence-design.md)와 함께 결정한다.

## 트랜잭션 경계 요약

| 작업 | DB 트랜잭션 범위 | 이벤트 발행 시점 | 재시도 |
| --- | --- | --- | --- |
| 찜 토글 | `interests` 단일 행 | 커밋 후 발행(확인 필요: outbox 도입 여부) | 낙관적 잠금 충돌 시 1회 |
| 조회 기록 | `drop_interest_counters` 단일 행 | 발행 없음(카운터 직접 갱신) | Redis 실패는 재시도, DB 실패는 확인 필요 |
| 오픈전 카운터 반영 | `drop_interest_counters` 단일 행 | 없음(내부 소비만) | 멱등키 확정 후 재시도 가능 |
| 랭킹 리스트 전환 | `drop_interest_counters` 단일 행 | `랭킹 리스트 전환됨` 발행 | 계약 확정 후 결정 |
| 오픈 후 점수 갱신 | `drop_interest_counters` 단일 행 | `오픈후 점수 갱신됨` 발행 | 계약 확정 후 결정 |

## 확인 필요

- 찜 추가/해제 이벤트를 트랜잭셔널 outbox로 발행할지, 커밋 후 직접 발행할지.
- Event Consumer들의 멱등키 설계(이벤트 ID 처리 이력 저장 위치와 보존 기간).
- `catalog.drop.updated` 계약이 `packages/contracts`에 언제 추가되는지 catalog 담당자와 협의.
- 당일 카운터 리셋을 배치로 할지 lazy reset으로 할지.
