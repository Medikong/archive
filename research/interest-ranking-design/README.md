# Interest & Ranking Design Research

국내/해외 기업의 찜(위시리스트)/실시간 인기 랭킹/조회수 집계 설계 사례를 공개 자료 기준으로 조사한 문서 묶음이다.

목표는 실제 회사 기술 블로그, 공식 발표 자료, 검증된 랭킹 알고리즘 원리에서 관심 신호(찜, 조회) 수집과 실시간 인기 랭킹 설계의 반복 패턴을 모으고, DropMong interest-service 설계에 참고할 수 있는 선택지를 정리하는 것이다. 이 문서 묶음은 조사와 분석을 위한 공간이며, 실제 적용 결정은 `REQ.A.07`(요구사항)과 후속 설계 문서에서 다룬다.

이 도메인은 결제/쿠폰 도메인만큼 공개된 장애 postmortem이 많지 않다. 1차 조사(국내 2개 회사)가 얇다고 판단해 국내 2개 회사와 해외 6개 회사/서비스, 실시간 랭킹 알고리즘 원리까지 범위를 넓혔다.

## 조사 결과 요약

- 국내 자료는 [무신사](https://www.musinsa.com/content/mz/6622)(실시간 랭킹 발표), [29CM](https://medium.com/29cm/api-v2-%EC%A0%84%ED%99%98%EA%B3%BC-db-%EB%AC%B4%EC%A4%91%EB%8B%A8-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-%ED%9B%84%EA%B8%B0-8b39eb0db566)(좋아요 API 마이크로서비스 분리), [크림](https://kream.co.kr/content/ranking_all-rising)(한정판 거래 플랫폼의 랭킹 신호), [네이버/다음](https://www.clien.net/service/board/park/5599879)(실시간 급상승 판정 원리)을 확인했다.
- 해외 자료는 커머스 postmortem 대신 **검증된 랭킹 알고리즘 원리**(Reddit, Hacker News)와 **가장 구체적인 조회수 측정 사례**(YouTube), 그리고 DropMong과 도메인이 동일한 한정판 드롭 앱(Nike SNKRS)을 확인했다. Mercari, Pinterest는 공식 엔지니어링 블로그의 존재만 확인했고 원문은 아직 열람하지 못했다 (상태: 후보).
- StockX/GOAT(스니커 리셀), 쿠팡 타임딜은 검색했지만 마케팅/소비자 팁 글 위주라 기술적 근거를 찾지 못해 제외했다 (`sources.md` 참고).
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
    global/
      README.md
      reddit/
        README.md
        posts/
      hackernews/
        README.md
        posts/
      mercari/
        README.md
        posts/
      nike-snkrs/
        README.md
        posts/
      pinterest/
        README.md
        posts/
      youtube/
        README.md
        posts/
  analysis/
    README.md
    01-view-count-dedup.md
    02-realtime-ranking-score-design.md
    03-view-count-measurement-methods.md
```

## 조사 단위

- 회사별 `README.md`: 회사의 찜/랭킹 관련 설계 특징을 요약한다.
- 회사별 출처 목록은 각 회사 `README.md`의 `확인한 공개 자료` 표와 루트 [sources.md](sources.md)에 함께 둔다.
- `posts/`: 공개 포스트를 스크랩한 본문을 둔다.
- `analysis/`: 특정 기업에 묶이지 않는 일반 구현 패턴이나, 여러 회사 자료를 주제별로 다시 묶는 문서를 둔다.

## 현재 범위

- 국내/해외 공개 자료 중 찜/좋아요 기능의 서비스 분리, 실시간 랭킹 발표·알고리즘 원리, 조회수 정의/중복 방지 구현 패턴을 조사한다.
- 개인화 추천 알고리즘(협업 필터링 등), 검색 랭킹(relevance), 유료 리포트로만 공개된 내부 지표는 이번 조사 범위 밖이다.
- 공개 자료에 없는 내부 구현(구체적 트래픽 규모, 실제 장애 이력, 정확한 상수값 등)은 추정하지 않는다. 필요하면 `확인 필요`로 남긴다.
- Mercari, Pinterest는 "후보" 상태로, 원문 전체를 아직 열람하지 못했다. 다음 조사에서 우선적으로 보강해야 한다.
