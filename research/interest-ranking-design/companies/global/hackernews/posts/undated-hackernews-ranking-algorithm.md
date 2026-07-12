# Reverse Engineering the Hacker News Ranking Algorithm

- 회사/저자: sangaline.com (외부 분석, HN 공식 발표 아닐 수 있음)
- 원문: https://sangaline.com/post/reverse-engineering-the-hacker-news-ranking-algorithm/
- 접근일: 2026-07-09
- 주제: 거듭제곱 시간 감쇠, gravity 상수
- 확인 방식: WebSearch 결과 스니펫 기준 (원문 전체 미열람)

## 원문 확인 내용

- Hacker News는 게시물 나이에 따라 점수를 거듭제곱 법칙(power law)으로 감쇠시킨다.
- 감쇠 상수는 약 2시간, gravity 계수는 약 0.75로 알려져 있다.
- Reddit의 "상대적으로 밀려남" 방식과 달리, HN은 나이에 따라 점수를 적극적으로 깎는 방식을 취한다.

## 적용 아이디어

- DropMong의 오픈 후 랭킹 공식(`sell_through_rate / elapsed_minutes`)은 선형 감쇠인데, 이를 거듭제곱 감쇠(`sell_through_rate / elapsed_minutes^gravity`, gravity < 1)로 바꾸면 오픈 초반 짧은 시간 동안의 급격한 순위 변동을 완화할 수 있다. 이는 `REQ.A.07` 열린 질문에 검토 항목으로 추가할 수 있다.
- 감쇠 상수를 고정값이 아니라 조정 가능한 파라미터로 설계한다는 접근 자체가, DropMong도 초기 상수(1분 바닥값 등)를 하드코딩하지 않고 설정값으로 분리해야 한다는 근거가 된다.

## 확인 필요

- 이 자료가 HN 공식 알고리즘 발표인지 외부 리버스 엔지니어링 추정인지 원문에서 명확히 확인해야 한다. 제목상 "Reverse Engineering"이라 완전히 공식화된 근거는 아닐 가능성이 있다.
