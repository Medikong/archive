# 핫키 카운터 동시성

## 질문

드롭 오픈 순간처럼 특정 상품 하나에 찜/조회 요청이 짧은 시간에 몰릴 때("핫키"), 카운터 증가 자체의 동시성을 어떻게 안전하고 빠르게 처리하는가?

## 원문에서 확인한 내용

- [Alibaba Cloud의 플래시 세일(스냅업) 시스템 사례](https://www.alibabacloud.com/blog/high-concurrency-practices-of-redis-snap-up-system_597858)는 요청을 클라이언트 차단 → 서버 로컬 캐시 → Redis → DB 순으로 단계적으로 걸러내는 계층형 아키텍처를 쓴다. 재고 차감은 Lua 스크립트로 "확인 후 감소"를 원자적으로 처리해 초과 판매를 막고, `SET NX EX` 기반 분산 락과 CAD(Compare And Delete)로 락 오작동을 방지한다.
- [Toss](https://toss.tech/article/27600)는 선착순류 카운팅에서 매 요청마다 Redis Increment를 부르지 않고, 로컬 캐시에서 먼저 카운팅한 뒤 일정 주기로 ScheduleJob이 Redis에 flush하는 방식으로 Redis CPU 부하를 줄인다. 이는 정확도(약간의 지연 허용)와 처리량을 맞바꾸는 접근이다.
- [Redis 공식 문서](https://redis.io/tutorials/howtos/leaderboard/)의 `ZINCRBY`는 read-modify-write 없이 원자적으로 점수를 증감시켜, 애플리케이션 레벨 락 없이도 동시 카운트 경쟁을 안전하게 처리한다.
- [Facebook의 좋아요 버튼 사례](https://medium.com/@V9vek/5-billion-likes-a-day-the-system-design-behind-facebooks-like-button-501e4bf1044d)는 `postId` 기준으로 `PostLikesCount` 저장소를 샤딩해 바이럴 게시물 하나의 폭증한 쓰기가 다른 게시물 처리에 영향을 주지 않게 격리하고, "즉시 쓰기(`Likes` 테이블) + Debezium(CDC)→Kafka→Worker 비동기 집계"의 2단계 구조로 정확성(방금 누른 좋아요는 즉시 반영)과 처리량(집계는 밀리초 지연 허용)을 동시에 만족시킨다.

## DropMong 적용 아이디어

- interest-service의 찜/조회 카운트는 Redis `ZINCRBY`(원자적 증감)를 기본으로 쓰되, 드롭 오픈 직후 짧은 시간(예: 수십 초~수 분) 동안 특정 인기 드롭 하나에 극단적으로 요청이 몰리는 경우를 대비해, Toss 사례처럼 애플리케이션 서버 로컬 카운터 + 주기적 flush로 완충하는 2단계 설계를 검토할 수 있다. 이는 팀의 쿠폰 서비스가 이미 쓰는 "Redis admission-gate + Postgres ledger" 패턴과도 같은 계열(짧은 시간 창 안의 트래픽 폭증을 앞단에서 흡수)이라 팀 내에서 설계를 공유하기 쉽다.
- 재고 차감(Alibaba 사례)은 "절대 정확해야 하는" 값이지만, 찜/조회 카운트는 랭킹 점수 계산용이라 약간의 지연/오차가 있어도 서비스 치명도가 낮다. 따라서 재고만큼 엄격한 Lua 원자적 처리나 분산 락까지는 필요 없고, `ZINCRBY` 수준의 원자성 + 필요 시 로컬 버퍼링으로 충분할 가능성이 크다 — "정확성보다 처리량"을 우선하는 방향으로 결정한다.
- 만약 특정 드롭 하나의 카운터가 실제로 병목이 되는 규모(단일 Redis 키에 초당 수만 건 이상의 쓰기)에 도달하면, Facebook 사례처럼 `dropId` 기준으로 격리하는 패턴(Redis Cluster를 쓰면 해시 슬롯 분산으로 어느 정도 자연스럽게 얻어짐)을 후속 확장으로 검토할 수 있다. 다만 1차 스코프에서는 이 정도 트래픽이 실제로 발생할지 불확실하므로, 지금 구현하지는 않고 향후 확장 여지로만 남긴다.
- "찜 상태(즉시 정확)"와 "랭킹 집계용 카운트(약간의 지연 허용)"를 다른 정합성 수준으로 다룬다는 원칙은 Toss 사례와 Facebook 사례(즉시 쓰기+비동기 집계) 양쪽에서 공통으로 확인돼, interest-service 설계의 기본 방향으로 채택할 만하다.

## 확인 필요

- interest-service의 실제 예상 트래픽(초당 찜/조회 요청 수)이 `ZINCRBY` 단일 키만으로 충분한 수준인지, 아니면 로컬 버퍼링/샤딩까지 필요한 수준인지는 실측 없이는 판단할 수 없다.
