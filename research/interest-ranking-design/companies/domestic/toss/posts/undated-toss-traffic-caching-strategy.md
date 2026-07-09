# 서버 증설 없이 처리하는 대규모 트래픽

- 회사: Toss
- 원문: https://toss.tech/article/27600
- 접근일: 2026-07-09
- 주제: Redis 캐싱 전략(Universal Data vs User-Specific Data), 카운터 로컬 집계 후 flush, API 통합을 통한 트래픽 감소
- 확인 방식: WebFetch로 원문 요약 확인 (Toss Tech 공식 블로그)

## 원문 확인 내용

- 토스 라이브 쇼핑처럼 피크 시간대 동시 접속자가 급증하는 서비스에서, 캐시 데이터를 "Universal Data"(모든 유저에게 동일 — 방송 리스트, 상세 정보)와 "User-Specific Data"(유저별로 다름 — 시청 여부 등)로 나눠 다르게 처리한다.
- Universal Data는 한 Redis 키에 GET 요청이 몰리면 Redis CPU 부하가 커지는 문제가 있어, 웹 서버의 로컬 캐시에 전부 올리고 변경 시 Redis Pub/Sub으로 각 서버의 로컬 캐시를 동기화한다.
- User-Specific Data는 사용자 수 증가에 따라 Redis 메모리가 폭증하는 문제가 있어, 데이터 압축으로 크기를 최소화한다.
- 포인트 지급 같은 중복 방지가 중요한 로직은 RedLock 기반 분산 락을 쓰고, Redis와 DB 모두에 기록해 Redis로 즉시 응답하되 DB로 최종 정합성을 보장한다.
- 대량 INSERT가 몰리는 상황은 Kafka로 비동기 처리하고, Consumer에서 Throttling을 걸어 DB가 감당 가능한 QPS를 넘지 않도록 조절한다.
- 선착순류 카운팅은 매 요청마다 Redis Increment를 부르지 않고, 로컬 캐시에서 먼저 카운팅한 후 특정 주기로 ScheduleJob이 Redis에 flush하는 방식으로 Redis CPU 부하를 줄인다.
- 초기에 분리돼 있던 조회 관련 API 3개를 하나(`/view`)로 통합해 피크 트래픽을 50% 감소시켰다.

## 적용 아이디어

- interest-service의 오픈 후 랭킹처럼 "모두에게 동일한 결과"(TOP N 랭킹)는 Universal Data로 분류해 애플리케이션 서버 로컬 캐시 + 짧은 TTL로 서빙하고, "내가 이 드롭을 찜했는지"처럼 사용자별 데이터는 별도 전략(Redis 개별 키 또는 DB)으로 분리하는 설계를 검토할 수 있다.
- 찜/조회 이벤트가 특정 인기 드롭에 몰릴 때, 매 요청마다 Redis `INCR`/`ZINCRBY`를 부르는 대신 짧은 시간 단위로 로컬에서 집계 후 배치로 반영하는 패턴은 오픈 직후 트래픽 스파이크 대응에 참고할 수 있다.

## 확인 필요

- 로컬 캐시 집계 방식의 구체적인 flush 주기, 서버 재시작/장애 시 유실 허용 범위는 원문 요약만으로는 확인하지 못했다. 실제 도입 시 DropMong 자체 기준으로 정해야 한다.
