# Alibaba Cloud

## 개요

- 구분: 해외(중국) 클라우드 벤더 공식 자료 — 플래시 세일(스냅업) 시스템의 고동시성 처리 사례
- 주요 자료: "High-Concurrency Practices of Redis: Snap-Up System" (Alibaba Cloud Community 공식 블로그)
- 조사 초점: 짧은 시간에 트래픽이 폭증하는 한정 수량 판매(플래시 세일)에서 핫키/재고 카운터 동시성을 어떻게 처리하는가 — DropMong의 "드롭 오픈" 순간과 도메인이 가장 가깝다

## 핵심 설계 포인트

| 주제 | 확인한 내용 | 출처 |
| --- | --- | --- |
| 계층형 필터링 | 클라이언트 차단 → 서버 로컬 캐시 → Redis → DB 순으로 요청을 단계적으로 걸러내 최종적으로 DB에 도달하는 요청을 최소화 | [High-Concurrency Practices of Redis: Snap-Up System](https://www.alibabacloud.com/blog/high-concurrency-practices-of-redis-snap-up-system_597858) |
| 원자적 재고 처리 | 재고 감소 시 Lua 스크립트로 "재고 확인 후 감소"를 원자적으로 처리해 음수(초과 판매)를 방지 | 위 출처 |
| 원자적 카운터 | TairString(Redis 호환) `exIncrBy`로 상한/하한 경계를 가진 카운터를 제공, 경계를 넘으면 자체적으로 실패 반환 | 위 출처 |
| 분산 락 | `SET NX EX`로 락을 구현하고, CAD(Compare And Delete)로 락을 건 주체만 해제 가능하게 해 오작동을 방지 | 위 출처 |

## 참고할 점

- DropMong의 "드롭 오픈" 순간은 이 글의 "스냅업(플래시 세일) 시작 시점"과 동일한 트래픽 패턴(오픈 전 대기 → 오픈 순간 폭증 → 이후 감소)이다. 계층형 필터링(클라이언트/로컬캐시/Redis/DB) 개념은 interest-service의 조회/찜 API에도 적용할 수 있다 — 특히 오픈 직후 몇 초간 몰리는 조회 요청을 애플리케이션 서버 로컬 캐시로 먼저 걸러내는 설계.
- `SET NX EX`로 락을 구현하는 패턴은 이미 조사한 조회수 dedup(`analysis/01-view-count-dedup.md`)의 `SETNX`/`EX` 방식과 원리가 같다 — 같은 Redis 원자적 연산이 "중복 방지"와 "동시성 제어" 두 문제 모두에 쓰이는 셈이다.
- 팀의 쿠폰 서비스가 이미 "Redis admission-gate + Postgres ledger" 패턴을 쓰고 있는데, 이 글의 계층형 필터링 + Lua 원자적 재고 차감 방식과 같은 계열의 해법이다. interest-service의 카운터 설계도 이 팀 내 기존 패턴과 일관되게 가져갈 수 있다.

## DropMong에 그대로 가져오면 안 되는 점

- 이 글은 "재고를 정확히 맞추는 것"(초과 판매 방지)이 목표인 시스템 사례다. interest-service의 찜/조회수 카운트는 정확히 맞지 않아도(약간의 오차) 서비스에 치명적이지 않으므로, 재고 차감만큼 엄격한 원자성/락 비용을 들일 필요는 없다. 오히려 Toss 사례처럼 정확도를 일부 양보하고 처리량을 높이는 방향이 interest-service에는 더 맞을 수 있다.

## 확인 필요

- Lua 스크립트의 실제 처리량 한계, TairString이 오픈소스 Redis에는 없는 Alibaba Cloud 전용 확장 기능이라는 점(오픈소스 Redis로 대체 시 `ZINCRBY`/`INCR`+한도 체크 조합으로 유사하게 구현해야 함)은 별도로 확인이 필요하다.

## 관련 문서

- `posts/undated-alibaba-cloud-snapup-concurrency.md`
