# Facebook (Meta)

## 개요

- 구분: 해외 대형 소셜 플랫폼 — 좋아요(Like) 카운터의 대규모 처리 사례
- 주요 서비스: 게시물 좋아요 카운터
- 조사 초점: 유명인/바이럴 게시물처럼 특정 리소스 하나에 좋아요가 폭증할 때의 샤딩된 카운터 구현

## 핵심 설계 포인트

| 주제 | 확인한 내용 | 출처 |
| --- | --- | --- |
| 카운터 저장소 | `PostLikesCount` MongoDB 클러스터에 `{postId, likes, likedBy}` 형태로 저장, 노드별 해시 인덱스로 읽기/쓰기 최적화 | [5 Billion Likes a Day: The System Design Behind Facebook's Like Button](https://medium.com/@V9vek/5-billion-likes-a-day-the-system-design-behind-facebooks-like-button-501e4bf1044d) |
| 핫 포스트 대응 | `postId` 기준으로 샤딩해, 바이럴 게시물 하나(예: 유명인 포스트)가 받는 폭증한 좋아요가 다른 게시물 처리에 영향을 주지 않게 격리 | 위 출처 |
| 2단계 쓰기 구조 | (1) `Likes` 테이블에 즉시 쓰기로 read-after-write 일관성 보장, (2) Debezium(CDC)→Kafka→Worker가 비동기로 `PostLikesCount`를 갱신 | 위 출처 |
| 지연 허용 | 비동기 집계 지연은 밀리초 수준으로, 처리량과 데이터 신선도 사이의 균형을 맞춘다 | 위 출처 |

## 참고할 점

- `postId` 단위 샤딩은 `analysis/06-hotkey-counter-concurrency.md`에서 다룬 "특정 드롭 하나에 쓰기가 몰리는 문제"에 대한 구체적인 해법 사례다. interest-service도 `dropId`를 샤드 키로 삼는 방식(또는 Redis Cluster의 해시 슬롯 분산)으로 유사하게 격리할 수 있다.
- "즉시 쓰기(사용자에게 보이는 값) + 비동기 집계(랭킹용 값)"로 분리하는 2단계 구조는, interest-service의 "내가 찜했는지 여부"(즉시 정확해야 함)와 "랭킹에 반영되는 찜 카운트"(약간의 지연 허용 가능)를 다른 정합성 수준으로 다뤄야 한다는 기존 결론(Toss 사례, `companies/domestic/toss/README.md`)과 같은 방향이다.

## DropMong에 그대로 가져오면 안 되는 점

- Facebook은 MongoDB 클러스터 자체를 `postId`로 샤딩하는 인프라 수준의 해법이다. interest-service는 이 정도 규모의 커스텀 샤딩 인프라를 처음부터 구축할 필요는 없고, Redis 단일 카운터 + 필요시 로컬 버퍼링(Toss 사례)으로 시작해 실측 후 확장 여부를 판단하는 게 맞다.

## 확인 필요

- 정확한 샤드 개수 결정 기준, Kafka Worker의 처리량 한계는 원문 요약 수준에서 확인하지 못했다.

## 관련 문서

- `posts/undated-facebook-like-button-sharded-counter.md`
