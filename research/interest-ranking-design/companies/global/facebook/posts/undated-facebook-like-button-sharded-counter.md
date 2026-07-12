# 5 Billion Likes a Day: The System Design Behind Facebook's Like Button

- 회사: Facebook (Meta)
- 원문: https://medium.com/@V9vek/5-billion-likes-a-day-the-system-design-behind-facebooks-like-button-501e4bf1044d
- 저자: Vivek Sharma
- 접근일: 2026-07-09
- 주제: 좋아요 카운터 샤딩, 핫 포스트 격리, CDC 기반 비동기 집계
- 확인 방식: WebFetch로 원문 요약 확인 (제3자 시스템 설계 해설 글, Facebook 공식 발표 아님)

## 원문 확인 내용

- 좋아요 카운터는 `PostLikesCount`라는 MongoDB 클러스터에 `{postId, likes, likedBy}` 형태로 저장되고, 노드별 해시 인덱스로 읽기/쓰기 성능을 최적화한다.
- 핵심은 `postId` 기준 샤딩이다. 메시 같은 유명인의 바이럴 포스트가 초당 수천 개의 좋아요를 받아도, 그 게시물의 카운터 행 하나가 병목이 되지 않게 여러 머신에 분산한다. 유명인 포스트와 일반 친구 게시물이 서로 다른 샤드에서 독립적으로 처리된다.
- 쓰기는 2단계로 나뉜다: (1) `Likes` 테이블에 즉시 기록해 read-after-write 일관성(방금 누른 좋아요가 바로 보임)을 보장하고, (2) Debezium(변경 데이터 캡처)이 이 변경을 감지해 Kafka로 흘려보내면 Worker가 비동기로 `PostLikesCount`를 갱신한다.
- 이 비동기 집계 지연은 밀리초 수준으로, 사용자 경험에 체감되는 지연 없이 처리량을 확보한다.

## 적용 아이디어

- interest-service의 찜/조회 카운터도 `dropId` 단위로 격리하는 개념을 적용할 수 있다. 다만 Facebook처럼 MongoDB 클러스터 자체를 샤딩할 필요는 없고, Redis Cluster를 쓴다면 `dropId`가 자연스럽게 해시 슬롯에 분산되므로 별도 샤딩 설계 없이도 어느 정도 격리 효과를 얻을 수 있다. 정말 특정 dropId 하나가 병목이 될 정도로 트래픽이 몰릴 때만 Toss식 로컬 버퍼링(`analysis/06`)을 추가로 검토한다.
- "즉시 쓰기 + 비동기 집계" 2단계 구조는 interest-service에도 적용 가능하다: "내가 찜을 눌렀다"는 사용자에게 즉시 반영(찜 상태 테이블에 즉시 쓰기)하되, 랭킹에 쓰이는 집계 카운트는 이벤트 스트림(Kafka)을 거쳐 약간의 지연을 두고 갱신해도 무방하다. 이는 `REQ.A.07`이 이미 `catalog.drop.updated` 등 이벤트 기반 연동을 전제로 하는 것과 자연스럽게 맞는다.

## 확인 필요

- 정확한 샤드 개수 산정 기준, 실제 Kafka Worker의 처리량 한계는 원문 요약만으로 확인하지 못했다. 제3자 해설 글이라 Facebook 내부의 실제 구현과 다를 수 있다.
