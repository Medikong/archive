# Hacker News

## 개요

- 구분: 해외 기술 콘텐츠 커뮤니티
- 주요 서비스: Hacker News 프론트페이지 랭킹
- 조사 초점: 거듭제곱(power law) 기반 시간 감쇠 랭킹 공식

## 핵심 설계 포인트

| 주제 | 확인한 내용 | 출처 |
| --- | --- | --- |
| 점수 감쇠 방식 | 글의 나이에 따라 점수가 거듭제곱 법칙(power law)으로 감쇠 | [Reverse Engineering the Hacker News Ranking Algorithm](https://sangaline.com/post/reverse-engineering-the-hacker-news-ranking-algorithm/) |
| 감쇠 계수 | 감쇠 상수 약 2시간, gravity 계수 약 0.75로 알려짐 | 위 출처 |
| Reddit과의 차이 | Reddit은 점수를 능동적으로 깎지 않는 방식(상대적 밀림)이지만, HN은 나이에 따라 점수 자체를 적극적으로 감쇠시킨다 | 검색 결과 종합 |

## 참고할 점

- **거듭제곱 감쇠(power law decay)**는 DropMong의 현재 공식(`elapsed_minutes`로 단순 나누는 선형 감쇠)보다 정교한 대안이다. 오픈 직후에는 감쇠가 느리고, 시간이 지날수록 감쇠가 빨라지는(혹은 그 반대) 곡선을 설계할 수 있다.
- "감쇠 상수를 실험적으로 조정 가능한 파라미터로 둔다"는 접근은, DropMong도 `elapsed_minutes` 바닥값(현재 1분)이나 감쇠 방식을 고정하지 않고 실측 후 조정 여지를 열어두는 근거로 쓸 수 있다.

## DropMong에 그대로 가져오면 안 되는 점

- HN의 감쇠 상수(2시간, gravity 0.75)는 뉴스 콘텐츠 소비 패턴에 맞춰진 값이라 DropMong의 "드롭 오픈 후 소진 속도"라는 전혀 다른 시간 스케일(분 단위, 짧으면 초 단위)에는 그대로 적용할 수 없다. 아이디어(거듭제곱 감쇠)만 차용하고 상수는 DropMong 자체 실측으로 정한다.

## 확인 필요

- 이 감쇠 공식이 HN 운영진의 공식 발표인지, 외부 개발자의 리버스 엔지니어링 추정인지 원문에서 구분해서 확인해야 한다 (출처 제목이 "Reverse Engineering"인 것으로 보아 공식 발표가 아닐 가능성이 있다).

## 관련 문서

- `posts/undated-hackernews-ranking-algorithm.md`
