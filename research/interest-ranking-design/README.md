# Interest & Ranking Design Research

국내/해외 기업의 찜(위시리스트)/실시간 인기 랭킹/조회수 집계 설계 사례를 공개 자료 기준으로 조사한 문서 묶음이다.

목표는 실제 회사 기술 블로그, 공식 발표 자료, 검증된 랭킹 알고리즘 원리에서 관심 신호(찜, 조회) 수집과 실시간 인기 랭킹 설계의 반복 패턴을 모으고, DropMong interest-service 설계에 참고할 수 있는 선택지를 정리하는 것이다. 이 문서 묶음은 조사와 분석을 위한 공간이며, 실제 적용 결정은 `REQ.A.07`(요구사항)과 후속 설계 문서에서 다룬다.

이 도메인은 결제/쿠폰 도메인만큼 공개된 장애 postmortem이 많지 않다. 1차 조사(국내 2개 회사)가 얇다고 판단해 국내 6개, 해외 6개 회사/서비스, 실시간 랭킹 알고리즘 원리까지 범위를 넓혔다. 2차 조사에서는 "점수를 어떻게 계산하는가"를 넘어 **찜 API 자체의 설계**, **랭킹을 어떻게 빠르게 서빙하는가**, **드롭 오픈 순간의 핫키 동시성**까지 다뤘다. 이 과정에서 "실시간 급상승" 대신 **당일(00:00 리셋) 누적 카운트**로 오픈 전 랭킹을 단순화하는 결정도 내렸다 (`REQ.A.07.FR-003`, `analysis/02` 참고).

실제 의사결정에 쓰이지 못한 자료(Pinterest, Nike SNKRS, Mercari, algomaster.io 등)는 회사별 폴더를 만드는 대신 `sources.md`에 "제외"로만 짧게 기록해, 무엇을 시도했고 왜 못 썼는지는 남기되 불필요한 분량은 쌓지 않기로 했다.

## 조사 결과 요약

- 국내 자료는 [무신사](https://www.musinsa.com/content/mz/6622)(실시간 랭킹 발표), [29CM](https://medium.com/29cm/api-v2-%EC%A0%84%ED%99%98%EA%B3%BC-db-%EB%AC%B4%EC%A4%91%EB%8B%A8-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-%ED%9B%84%EA%B8%B0-8b39eb0db566)(좋아요 API 마이크로서비스 분리), [크림](https://kream.co.kr/content/ranking_all-rising)(한정판 거래 플랫폼의 랭킹 신호), [네이버/다음](https://www.clien.net/service/board/park/5599879)(실시간 급상승 판정 원리), [Toss](https://toss.tech/article/27600)(대규모 트래픽 캐싱/카운터 전략), [쿠팡](https://medium.com/coupang-engineering/%EB%8C%80%EC%9A%A9%EB%9F%89-%ED%8A%B8%EB%9E%98%ED%94%BD-%EC%B2%98%EB%A6%AC%EB%A5%BC-%EC%9C%84%ED%95%9C-%EC%BF%A0%ED%8C%A1%EC%9D%98-%EB%B0%B1%EC%97%94%EB%93%9C-%EC%A0%84%EB%9E%B5-184f7fdb1367)(마이크로서비스 데이터 통합 서빙)를 확인했다.
- 해외 자료는 커머스 postmortem 대신 **검증된 랭킹 알고리즘 원리**(Reddit, Hacker News), **가장 구체적인 조회수 측정 사례**(YouTube), **랭킹 서빙 표준 패턴**(Redis 공식 리더보드 튜토리얼), **플래시 세일 동시성 처리 사례**(Alibaba Cloud), **핫키 샤딩 카운터 사례**(Facebook)를 확인했다.
- 찜 API 설계는 특정 대기업 공식 블로그보다, 올리브영/무신사/컬리 세 회사의 실제 요청을 비교한 개인 기술 블로그가 가장 구체적이었다. 컬리의 PUT 기반 멱등 설계가 참고할 만하다.
- Nike SNKRS(한정판 드롭 도메인과 가장 가까움)는 읽어봤지만 서드파티 해설뿐이라 실제 요구사항에 인용되지 못했다. Pinterest/Mercari는 원문까지 열람했거나 시도했지만 구체적인 내용이 없거나 확인하지 못해 제외했다. StockX/GOAT, algomaster.io도 같은 이유로 제외했다 (`sources.md` 참고).
- 조회수 중복 집계 방지는 특정 기업 사례보다 Redis `SETNX`/`EX` 기반의 일반 구현 패턴으로 여러 개발자 블로그에서 반복 확인되어 `analysis/01-view-count-dedup.md`에 별도로 정리했다.
- `archive/바운디드-컨텍스트-분석.md`(올리브영/컬리 조사 풀)는 쿠폰/주문/알림/품절 시나리오 중심이라 찜/랭킹 주제는 다루지 않는다. 이번 조사가 이 영역의 첫 번째 조사 풀이다.

## 읽는 순서

| 문서 | 용도 |
| --- | --- |
| [sources.md](sources.md) | 수집한 출처 목록 (분석/후보/제외 상태 포함) |
| [taxonomy.md](taxonomy.md) | 찜/랭킹 설계 관점 분류 기준 |
| [companies/domestic/README.md](companies/domestic/README.md) | 국내 기업별 조사 인덱스 |
| [companies/global/README.md](companies/global/README.md) | 해외 기업/서비스별 조사 인덱스 |
| [analysis/README.md](analysis/README.md) | 주제별 2차 분석 인덱스 |

## 빠른 결론

| 주제 | 확인한 패턴 | DropMong 설계 메모 |
| --- | --- | --- |
| 기능 분리 시점 | ["좋아요" 기능을 모놀리식에 두었다가 나중에 마이크로서비스로 옮기면 무중단 마이그레이션 비용이 크다](https://medium.com/29cm/api-v2-%EC%A0%84%ED%99%98%EA%B3%BC-db-%EB%AC%B4%EC%A4%91%EB%8B%A8-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-%ED%9B%84%EA%B8%B0-8b39eb0db566). | 찜 기능을 user-service/catalog-service에 얹지 않고 처음부터 interest-service로 독립시킨다. |
| 실시간 인기 지표의 신뢰성 | [실시간 활동 지표 기반 랭킹은 어뷰징/조작 없는 공정성을 핵심 가치로 내세운다](https://www.musinsa.com/content/mz/6622). 네이버/다음은 신뢰성 논란으로 서비스를 중단한 이력이 있다. | 조회 신호는 로그인 사용자만 집계해 최소 진입장벽을 두고, 랭킹 로직의 한계를 문서로 명시한다. |
| 조회수 중복 집계 | [Redis `SETNX`/`EX`로 사용자·리소스 단위 시간 윈도우 dedup을 적용하는 패턴이 일반적이다](https://velog.io/@horang12/%EB%8F%99%EC%9D%BC%ED%95%9C-%EC%9C%A0%EC%A0%80%EC%9D%98-%EC%A1%B0%ED%9A%8C%EC%88%98%EA%B0%80-%EC%A4%91%EB%B3%B5-%EC%A7%91%EA%B3%84-%EB%90%98%EC%A7%80-%EC%95%8A%EA%B2%8C-%ED%95%98%EB%A0%A4%EB%A9%B4). [YouTube](https://uppbeat.io/blog/youtube/how-does-youtube-count-views)는 "일단 집계 후 사후 검증" 방식과 사용자별 24시간 제한을 함께 쓴다. | `SET NX EX 300 "viewed:{dropId}:{userId}"` 키로 5분 dedup을 적용한다 (사전 차단 방식 유지). |
| 실시간 랭킹 점수 설계 | [Reddit](https://medium.com/hacking-and-gonzo/how-reddit-ranking-algorithms-work-ef111e33d0d9)은 로그 스케일+상대적 밀림, [Hacker News](https://sangaline.com/post/reverse-engineering-the-hacker-news-ranking-algorithm/)는 거듭제곱 시간 감쇠를 쓴다. | 오픈 후 랭킹의 선형 감쇠(`elapsed_minutes`)를 거듭제곱 감쇠로 바꾸는 안을 열린 질문으로 검토한다. |
| 다중 신호 조합 | [크림](https://kream.co.kr/content/ranking_all-rising)은 조회+관심+거래를 조합하고 기간별(급상승/월간) 랭킹을 분리한다. | DropMong의 "찜+조회수"(오픈 전) / "소진율"(오픈 후) 이원화 설계와 같은 방향임을 재확인. |
| 찜 API 설계/멱등성 | [올리브영/무신사/컬리](https://beta.velog.io/@sweet_sumin/%EC%A2%8B%EC%95%84%EC%9A%94-%EB%A9%B1%EB%93%B1%EC%84%B1-%EC%98%AC%EB%A6%AC%EB%B8%8C%EC%98%81-%EB%AC%B4%EC%8B%A0%EC%82%AC-%EC%BB%AC%EB%A6%AC%EB%8A%94-%EC%96%B4%EB%96%BB%EA%B2%8C-%EB%8B%A4%EB%A5%B4%EA%B2%8C-%EA%B5%AC%ED%98%84%ED%96%88%EC%9D%84%EA%B9%8C)는 각기 다른 엔드포인트/메서드로 찜 멱등성을 처리하며, 컬리의 PUT 방식이 가장 단순하다. | interest-service 찜 API를 `PUT /interests/drops/{dropId}` 형태로 설계하는 안을 검토한다. |
| 랭킹 서빙 계층 | [Redis 공식 튜토리얼](https://redis.io/tutorials/howtos/leaderboard/)은 Sorted Set(`ZADD`/`ZINCRBY`/`ZREVRANGE`)을 표준 해법으로 제시한다. | 오픈 전/오픈 후 랭킹 모두 Sorted Set에 점수를 쓰고 TOP N을 조회하는 서빙 계층으로 구현하는 안을 검토한다. |
| 드롭 오픈 순간 핫키 동시성 | [Alibaba Cloud](https://www.alibabacloud.com/blog/high-concurrency-practices-of-redis-snap-up-system_597858)는 계층형 필터링+Lua 원자적 처리, [Toss](https://toss.tech/article/27600)는 로컬 집계 후 배치 flush로 대응한다. | 찜/조회 카운트는 재고만큼 엄격할 필요가 없어 Toss식 "로컬 집계+배치 flush"가 더 적합할 가능성이 크다. |

## 폴더 구조

```text
interest-ranking-design/
  README.md
  sources.md
  taxonomy.md
  companies/
    domestic/
      README.md
      musinsa/
        README.md
        posts/
      29cm/
        README.md
        posts/
      kream/
        README.md
        posts/
      naver-daum-trending/
        README.md
        posts/
      toss/
        README.md
        posts/
      coupang/
        README.md
        posts/
    global/
      README.md
      reddit/
        README.md
        posts/
      hackernews/
        README.md
        posts/
      youtube/
        README.md
        posts/
      redis/
        README.md
        posts/
      alibaba-cloud/
        README.md
        posts/
      facebook/
        README.md
        posts/
  analysis/
    README.md
    01-view-count-dedup.md
    02-realtime-ranking-score-design.md
    03-view-count-measurement-methods.md
    04-wishlist-api-design.md
    05-realtime-leaderboard-serving.md
    06-hotkey-counter-concurrency.md
```

## 조사 단위

- 회사별 `README.md`: 회사의 찜/랭킹 관련 설계 특징을 요약한다.
- 회사별 출처 목록은 각 회사 `README.md`의 `확인한 공개 자료` 표와 루트 [sources.md](sources.md)에 함께 둔다.
- `posts/`: 공개 포스트를 스크랩한 본문을 둔다.
- `analysis/`: 특정 기업에 묶이지 않는 일반 구현 패턴이나, 여러 회사 자료를 주제별로 다시 묶는 문서를 둔다.

## 현재 범위

- 국내/해외 공개 자료 중 찜/좋아요 기능의 서비스 분리·API 설계, 실시간 랭킹 발표·알고리즘 원리·서빙 계층, 조회수 정의/중복 방지 구현 패턴, 핫키/카운터 동시성 처리를 조사한다.
- 개인화 추천 알고리즘(협업 필터링 등), 검색 랭킹(relevance), 유료 리포트로만 공개된 내부 지표는 이번 조사 범위 밖이다.
- 공개 자료에 없는 내부 구현(구체적 트래픽 규모, 실제 장애 이력, 정확한 상수값 등)은 추정하지 않는다. 필요하면 `확인 필요`로 남긴다.
- Pinterest, Nike SNKRS, Mercari, algomaster.io는 확인했으나 실제 결정에 반영하지 못해 회사 폴더를 만들지 않고 `sources.md`에 "제외"로만 기록했다. 다음 조사가 필요해지면 그 표의 URL부터 다시 확인하면 된다.
