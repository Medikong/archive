# Redis (공식)

## 개요

- 구분: 인메모리 데이터스토어 벤더 공식 자료 (특정 커머스 회사는 아니지만, 실시간 랭킹 구현의 표준 패턴을 제공하는 1차 출처)
- 주요 자료: Redis 공식 튜토리얼 "Build a Real-Time Leaderboard with Redis Sorted Sets"
- 조사 초점: Sorted Set으로 실시간 랭킹/리더보드를 서빙하는 표준 구현 패턴

## 핵심 설계 포인트

| 주제 | 확인한 내용 | 출처 |
| --- | --- | --- |
| 자료구조 | Sorted Set은 모든 멤버가 점수를 가지며 Redis가 항상 점수 순으로 정렬된 상태를 유지 | [Build a Real-Time Leaderboard with Redis Sorted Sets](https://redis.io/tutorials/howtos/leaderboard/) |
| 쓰기 | `ZADD`로 점수 등록/갱신, `ZINCRBY`로 read-modify-write 경쟁 없이 원자적으로 점수 증감 | 위 출처 |
| 읽기 | `ZREVRANGE`로 TOP N/특정 구간 조회, `ZREVRANK`로 특정 멤버의 순위를 전체 스캔 없이 조회 | 위 출처 |
| 성능 특성 | 삽입/갱신은 O(log N), 구간 조회는 O(log N + M) — 수백만 멤버 규모에서도 밀리초 이내 | 위 출처 |
| 설계 철학 | "정렬 비용을 쓰기 시점에 지불한다"(SQL의 `ORDER BY`가 읽기 시점에 정렬하는 것과 대조) — 읽기가 쓰기보다 훨씬 많은 리더보드에 유리 | 위 출처 |

## 참고할 점

- interest-service의 오픈 전/오픈 후 랭킹 모두 "TOP N을 자주 읽고, 점수는 이벤트가 발생할 때만 갱신"되는 전형적인 읽기 편중 워크로드다. Redis Sorted Set은 점수 계산 로직(찜+조회수 합산, 소진율/시간)의 결과값을 쓰기만 하면 정렬/순위 조회를 자체적으로 처리해주므로, 랭킹 서빙 계층을 애플리케이션에서 직접 정렬하지 않아도 된다.
- `ZINCRBY`가 원자적 증감을 보장한다는 점은, 동시에 여러 사용자가 같은 드롭을 찜/조회할 때 애플리케이션 레벨 락 없이도 카운트 경쟁을 안전하게 처리할 수 있다는 근거가 된다(`analysis/06-hotkey-counter-concurrency.md` 참고).

## DropMong에 그대로 가져오면 안 되는 점

- Sorted Set은 "점수"만 관리하고, 점수 계산 공식(`REQ.A.07`의 소진율/시간, 찜+조회수 합산) 자체는 애플리케이션이 별도로 계산해서 넣어줘야 한다. Redis가 랭킹 알고리즘 자체를 대신 설계해주지는 않는다.
- 단일 Redis 인스턴스/키에 쓰기가 과도하게 몰리는 핫키 상황(오픈 순간 특정 드롭 하나에 폭증하는 찜/조회)은 Sorted Set 자체로 해결되지 않는다. 별도의 핫키 대응(로컬 버퍼링, 샤딩)이 필요하다.

## 확인 필요

- 대규모 동시 `ZINCRBY` 상황에서의 실제 처리량 한계, 클러스터 모드에서 Sorted Set 키 하나가 한 노드에 고정되는 데 따른 확장성 한계는 이번 조사에서 확인하지 못했다.

## 관련 문서

- `posts/undated-redis-official-leaderboard-tutorial.md`
