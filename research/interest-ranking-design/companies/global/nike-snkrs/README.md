# Nike SNKRS

## 개요

- 구분: 해외 한정판 스니커즈 "드롭" 커머스 앱 (DropMong과 도메인이 사실상 동일)
- 주요 서비스: Nike SNKRS 앱
- 조사 초점: 드롭 오픈 순간의 수요 처리 방식과 공정성 정책 — 회사 기술 블로그보다는 제품/정책 설명 위주

## 핵심 설계 포인트

| 주제 | 확인한 내용 | 출처 |
| --- | --- | --- |
| 드롭 방식 다양화 | FLOW(선착순), LEO(짧은 응모창+무작위 추첨), DAN(긴 응모창+완전 추첨) 세 가지 방식을 혼용 | [Nike's Drop Methods Explained](https://stashed-sneakers.com/blogs/blog-posts/nike-snkrs-drop-methods-guide-stashed), [How to Win on the Nike SNKRS App](https://captaincreps.com/how-to-cop-shoes-on-the-nike-snkrs-app/) |
| 응모 심사 기준 | 선착순이 아닌 방식에서는 구매 이력, 앱 참여도 등 "진짜 관심"을 나타내는 신호를 고려한다고 알려져 있으나 정확한 기준은 비공개 | 위 출처 |
| 수요 예측 | Celect(예측 분석 기업) 인수로 재고 예측 주기를 6개월에서 30분 단위로 단축했다고 알려짐 | [Nike's Advanced Demand Sensing](https://sparkco.ai/blog/nikes-advanced-demand-sensing-beyond-excel-forecasts) |

## 참고할 점

- **드롭 방식을 인기도/희소성에 따라 다르게 운영**하는 아이디어(선착순 vs 추첨)는, DropMong도 모든 드롭에 동일한 랭킹/판매 로직을 적용하기보다 향후 드롭 유형별 정책 분리를 고려할 근거가 된다. (단, 이번 `REQ.A.07` 스코프는 랭킹 산출만 다루고 판매 방식 자체는 범위 밖이다.)
- "선착순 대신 진짜 관심 신호(구매 이력, 앱 참여도)를 반영한다"는 접근은, DropMong의 찜/조회수 기반 오픈 전 랭킹이 단순 클릭 경쟁이 아니라 관심의 질을 반영하려는 방향과 같은 문제의식이다.

## DropMong에 그대로 가져오면 안 되는 점

- Nike SNKRS의 정확한 심사 알고리즘은 비공개다. "관심 신호를 반영한다"는 방향성만 참고할 수 있고, 구체적인 가중치나 판정 로직은 알 수 없다.
- Celect 기반 수요 예측(6개월→30분)은 대형 브랜드의 자체 AI 인프라 이야기라 DropMong 초기 스코프와는 규모 차이가 크다.

## 확인 필요

- 이 조사는 엔지니어링 블로그가 아니라 소비자 대상 설명 글/서드파티 분석 위주라, 실제 내부 구현은 대부분 "확인 필요"로 남는다.

## 관련 문서

- `posts/undated-nike-snkrs-drop-mechanics.md`
