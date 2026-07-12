# 실시간 랭킹 서빙 계층 설계

## 질문

`analysis/02-realtime-ranking-score-design.md`가 "점수를 어떤 공식으로 계산하는가"를 다뤘다면, 이 문서는 "계산된 점수로 TOP N 랭킹을 어떻게 빠르게 조회 가능한 형태로 서빙하는가"를 다룬다.

## 원문에서 확인한 내용

- [Redis 공식 리더보드 튜토리얼](https://redis.io/tutorials/howtos/leaderboard/)은 Sorted Set을 표준 해법으로 제시한다. `ZADD`/`ZINCRBY`로 점수를 쓰고, `ZREVRANGE`로 TOP N을, `ZREVRANK`로 특정 멤버의 순위를 조회한다. 삽입/갱신은 O(log N), 범위 조회는 O(log N + M)로 수백만 멤버 규모에서도 밀리초 이내 응답한다.
- Redis의 설계 철학은 "정렬 비용을 쓰기 시점에 지불한다"는 것이다. SQL의 `ORDER BY`는 매 읽기마다 정렬하지만, Sorted Set은 `ZADD` 시점에 정렬 상태를 유지해 읽기가 훨씬 많은 랭킹류 워크로드에 유리하다.
- [크림(KREAM)](https://kream.co.kr/content/ranking_all-rising)은 급상승(최근 기간)과 월간 랭킹을 분리해 제공한다 — 이는 서빙 관점에서 보면 "같은 원본 이벤트를 다른 시간 윈도우로 집계한 두 개의 별도 Sorted Set(또는 랭킹 키)을 유지한다"는 의미로 해석할 수 있다.
- [Toss](https://toss.tech/article/27600)는 모두에게 동일한 데이터("Universal Data")를 Redis 대신 애플리케이션 서버의 로컬 캐시에 올리고 Pub/Sub으로 동기화하는 방식을 쓴다.

## DropMong 적용 아이디어

- interest-service의 랭킹 저장소로 Redis Sorted Set을 채택하는 안을 검토한다: 오픈 전 랭킹은 `ranking:pre-open` 키에 `score = 찜수+조회수 합산(또는 로그 스케일 변형)`, 오픈 후 랭킹은 `ranking:hot` 키에 `score = 소진율/시간` 공식 결과를 저장한다. 찜/조회 이벤트가 들어올 때마다 전체 목록을 다시 정렬하지 않고 `ZINCRBY`/`ZADD` 한 번으로 순위를 갱신할 수 있다.
- TOP N 랭킹 API(`GET /rankings/pre-open`, `GET /rankings/hot`) 응답은 Toss 사례처럼 애플리케이션 서버 로컬 캐시에 수 초~수십 초 TTL로 캐싱해, 랭킹 조회 트래픽이 몰려도 Redis에 직접 부하를 주지 않게 한다. 랭킹은 "지금 이 순간의 정확한 값"보다 "대략적으로 최신인 값"이면 충분한 데이터이므로 이 정도의 지연은 허용 가능하다.
- 오픈 후 랭킹처럼 분자(확정 수량)/분모(총 수량)가 계속 바뀌는 점수는 이벤트가 올 때마다 공식을 다시 계산해 `ZADD`로 값을 덮어써야 한다(`ZINCRBY`처럼 증분만으로는 처리 불가). 이 재계산 비용이 오픈 직후 트래픽 스파이크와 겹치므로, 재계산 주기(이벤트마다 즉시 vs 짧은 배치 주기)를 결정해야 한다 — 열린 질문으로 남긴다.

## 확인 필요

- Redis 클러스터 모드에서 하나의 Sorted Set 키가 특정 노드에 고정되는 데 따른 확장 한계, 그리고 오픈 후 랭킹처럼 짧은 주기로 점수를 재계산해 덮어쓰는 방식의 실제 처리량은 이번 조사에서 확인하지 못했다. DropMong 자체 부하 테스트가 필요하다.
