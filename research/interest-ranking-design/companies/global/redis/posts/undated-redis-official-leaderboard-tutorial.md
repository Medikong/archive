# Build a Real-Time Leaderboard with Redis Sorted Sets

- 회사: Redis (공식)
- 원문: https://redis.io/tutorials/howtos/leaderboard/
- 저자: Ajeet Raina
- 접근일: 2026-07-09
- 주제: Sorted Set 기반 실시간 리더보드 구현, ZADD/ZINCRBY/ZREVRANGE/ZREVRANK
- 확인 방식: WebFetch로 원문 요약 확인 (redis.io 공식 튜토리얼)

## 원문 확인 내용

- Redis Sorted Set의 모든 멤버는 부동소수점 점수를 가지며, Redis는 항상 점수 순으로 정렬된 상태를 유지한다.
- `ZADD companyLeaderboard 2600000000000 company:AAPL`처럼 멤버와 점수를 등록/갱신한다.
- `ZINCRBY companyLeaderBoard 1000000000 "company:FB"`로 read-modify-write 없이 원자적으로 점수를 증감한다.
- `ZREVRANGE companyLeaderboard 0 9 WITHSCORES`로 TOP 10을 조회하고, `ZCOUNT`로 특정 점수 구간에 속한 멤버 수를 센다.
- 삽입/갱신은 O(log N), 범위 조회는 O(log N + M)이며, 5천만 멤버 규모에서도 1ms 미만으로 동작한다고 설명한다.
- 핵심 설계 철학: "정렬 작업을 쓰기 시점에 한다" — SQL의 `ORDER BY`가 읽기 시점에 정렬하는 것과 달리, Redis는 `ZADD` 시점에 정렬 비용을 지불한다. 읽기가 쓰기보다 훨씬 많은 리더보드 워크로드에 적합하다.

## 적용 아이디어

- interest-service의 랭킹 저장소로 Sorted Set을 검토할 수 있다: 오픈 전 랭킹은 `ZADD ranking:pre-open {score} {dropId}`(score = 찜+조회수 합산 또는 로그 스케일 변형), 오픈 후 랭킹은 `ZADD ranking:hot {score} {dropId}`(score = 소진율/시간 공식 결과)로 관리하고, TOP N 조회는 `ZREVRANGE`로 처리하는 방식이다.
- 찜/조회 이벤트가 들어올 때마다 애플리케이션이 전체 목록을 다시 정렬하지 않고 `ZINCRBY`/`ZADD` 한 번으로 순위를 갱신할 수 있어, `analysis/02-realtime-ranking-score-design.md`에서 검토한 점수 공식들을 실제로 서빙하는 계층으로 자연스럽게 이어진다.

## 확인 필요

- 이 튜토리얼은 정적인 시가총액 예시를 다뤄, DropMong처럼 짧은 시간 안에 점수가 계속 변하는 시나리오(오픈 후 소진율은 분모/분자가 계속 바뀜)에서 매번 점수를 다시 계산해 `ZADD`로 덮어써야 하는지, 아니면 증분만 반영 가능한지는 별도로 설계해야 한다.
