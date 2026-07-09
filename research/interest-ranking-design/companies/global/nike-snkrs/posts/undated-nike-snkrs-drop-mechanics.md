# Nike SNKRS 드롭 방식과 수요 처리

- 회사: Nike
- 원문: https://stashed-sneakers.com/blogs/blog-posts/nike-snkrs-drop-methods-guide-stashed , https://captaincreps.com/how-to-cop-shoes-on-the-nike-snkrs-app/ , https://sparkco.ai/blog/nikes-advanced-demand-sensing-beyond-excel-forecasts
- 접근일: 2026-07-09
- 주제: 드롭 방식(FLOW/LEO/DAN), 응모 심사, 수요 예측
- 확인 방식: WebSearch 결과 스니펫 기준. 서드파티(스니커 커뮤니티/컨설팅) 글이라 Nike 공식 기술 블로그는 아님.

## 원문 확인 내용

- SNKRS는 세 가지 드롭 방식을 쓴다: FLOW(전통적 선착순, 일반 발매/깜짝 재입고용), LEO(2~3분의 짧은 응모창 후 무작위 추첨, 중간 정도 인기 발매용), DAN(10~30분의 긴 응모창, 완전 추첨, 초고인기 한정 발매용).
- 추첨 방식에서는 구매 이력, 앱 참여도 등 "진짜 관심"을 나타내는 신호가 반영될 수 있다고 알려져 있지만, Nike는 정확한 기준을 공개하지 않는다.
- Nike는 예측 분석 기업 Celect를 인수해 재고 예측 주기를 6개월에서 30분 단위로 단축했다고 알려졌다.

## 적용 아이디어

- DropMong도 모든 드롭에 동일한 판매 방식을 적용하지 않고, 인기도에 따라 판매/참여 방식을 다르게 가져갈 수 있다는 아이디어의 근거로 쓸 수 있다 (다만 이는 `REQ.A.07`의 스코프 밖이며, 향후 order-service나 별도 waiting-room-service 논의로 넘길 사안이다).
- "구매 이력/참여도 같은 관심의 질을 반영한다"는 철학은, DropMong의 찜/조회수 기반 오픈 전 랭킹 설계 방향과 일치한다.

## 확인 필요

- Nike 공식 기술 블로그가 아닌 서드파티 해설 글이라, 실제 알고리즘 구현 여부와 상세는 확인할 수 없다. Nike는 알고리즘을 비공개로 유지한다고 알려져 있다.
