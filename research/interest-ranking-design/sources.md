# Sources

찜(위시리스트)/실시간 인기 랭킹/조회수 집계 조사에 사용한 공개 출처 목록이다.
긴 원문을 복제하지 않고, 원문 링크와 확인 내용 요약을 회사별 스크랩 노트와 분석 문서에 나누어 남긴다.

접근일은 모두 `2026-07-09` 기준이다.

| 상태 | 구분 | 회사/저자 | 제목 | URL | 접근일 | 언어 | 주제 | 저장 위치 | 비고 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 분석 | 국내 | 무신사 | 무신사 스토어, 국내 최초 실시간 베스트 랭킹 서비스 제공 | https://www.musinsa.com/content/mz/6622 | 2026-07-09 | ko | 실시간 랭킹, 어뷰징 방지, 공정성 | companies/domestic/musinsa/posts/undated-musinsa-realtime-ranking.md | 검색 스니펫 기준 확인, 원문 전체 스크랩 아님 |
| 분석 | 국내 | 29CM | API V2 전환과 DB 무중단 마이그레이션 후기 | https://medium.com/29cm/api-v2-%EC%A0%84%ED%99%98%EA%B3%BC-db-%EB%AC%B4%EC%A4%91%EB%8B%A8-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-%ED%9B%84%EA%B8%B0-8b39eb0db566 | 2026-07-09 | ko | 좋아요 API, 마이크로서비스 분리, 무중단 마이그레이션, SQS 이중 기록 | companies/domestic/29cm/posts/undated-29cm-like-api-migration.md | 검색 스니펫 기준 확인, 원문 전체 스크랩 아님 |
| 분석 | 개인 블로그 | horang12 (velog) | 동일한 유저의 조회수가 중복 집계 되지 않게 하려면? | https://velog.io/@horang12/%EB%8F%99%EC%9D%BC%ED%95%9C-%EC%9C%A0%EC%A0%80%EC%9D%98-%EC%A1%B0%ED%9A%8C%EC%88%98%EA%B0%80-%EC%A4%91%EB%B3%B5-%EC%A7%91%EA%B3%84-%EB%90%98%EC%A7%80-%EC%95%8A%EA%B2%8C-%ED%95%98%EB%A0%A4%EB%A9%B4 | 2026-07-09 | ko | Redis SETNX/TTL, 조회수 중복 방지 | analysis/01-view-count-dedup.md | 특정 기업 사례 아님, 일반 구현 패턴 |
| 참고 | 개인 블로그 | 10000ji_ (velog) | [재능교환소] 조회수 중복 방지 (Redis, 쿠키) | https://velog.io/@10000ji_/%EC%9E%AC%EB%8A%A5%EA%B5%90%ED%99%98%EC%86%8C-%EC%A1%B0%ED%9A%8C%EC%88%98-%EC%A4%91%EB%B3%B5-%EB%B0%A9%EC%A7%80-Redis-%EC%BF%A0%ED%82%A4 | 2026-07-09 | ko | 로그인/비로그인 분리 dedup | analysis/01-view-count-dedup.md | 로그인은 Redis, 비로그인은 쿠키로 분리하는 패턴 참고용 |
| 분석 | 국내 | 크림(KREAM) | 급상승 전체 랭킹 / 월간 전체 랭킹 | https://kream.co.kr/content/ranking_all-rising , https://kream.co.kr/content/ranking_all-monthly | 2026-07-09 | ko | 랭킹 신호(조회+관심+거래), 기간별 랭킹 분리 | companies/domestic/kream/posts/undated-kream-ranking-signals.md | 제품 화면 기준, 엔지니어링 블로그 아님 |
| 분석 | 국내 | 네이버/다음 | 실시간 급상승 검색어 알고리즘 설명 + KISO 검증보고서 | https://www.clien.net/service/board/park/5599879 , https://www.kiso.or.kr/wp-content/uploads/2014/10/3%EC%B0%A8-%EA%B2%80%EC%A6%9D%EB%B3%B4%EA%B3%A0%EC%84%9C.pdf | 2026-07-09 | ko | 평균 대비 증가율 판정, 알고리즘 신뢰성 검증 | companies/domestic/naver-daum-trending/posts/undated-naver-daum-trending-algorithm.md | KISO 보고서 원문 PDF는 미열람 |
| 분석 | 해외 | Reddit | How Reddit ranking algorithms work | https://medium.com/hacking-and-gonzo/how-reddit-ranking-algorithms-work-ef111e33d0d9 | 2026-07-09 | en | 로그 스케일 득표, 시간 가중치 | companies/global/reddit/posts/undated-reddit-hot-ranking-algorithm.md | 업계에서 널리 인용되는 원출처 |
| 분석 | 해외 | Hacker News | Reverse Engineering the Hacker News Ranking Algorithm | https://sangaline.com/post/reverse-engineering-the-hacker-news-ranking-algorithm/ | 2026-07-09 | en | 거듭제곱 시간 감쇠 | companies/global/hackernews/posts/undated-hackernews-ranking-algorithm.md | 공식 발표 아닌 외부 리버스 엔지니어링 가능성 |
| 후보 | 해외 | Mercari | Zero Downtime Migration with Strong Data Consistency Part II | https://engineering.mercari.com/en/blog/entry/20241113-designing-a-zero-downtime-migration-solution-with-strong-data-consistency-part-ii/ | 2026-07-09 | en | 무중단 마이그레이션, 데이터 정합성 | companies/global/mercari/posts/undated-mercari-zero-downtime-migration.md | 제목/존재만 확인, 원문 미열람 |
| 분석 | 해외 | Nike SNKRS | 드롭 방식(FLOW/LEO/DAN) 및 응모 심사 해설 | https://stashed-sneakers.com/blogs/blog-posts/nike-snkrs-drop-methods-guide-stashed , https://captaincreps.com/how-to-cop-shoes-on-the-nike-snkrs-app/ , https://sparkco.ai/blog/nikes-advanced-demand-sensing-beyond-excel-forecasts | 2026-07-09 | en | 드롭 방식, 관심 신호 기반 심사, 수요 예측 | companies/global/nike-snkrs/posts/undated-nike-snkrs-drop-mechanics.md | 서드파티 해설 글, Nike 공식 기술 블로그 아님 |
| 후보 | 해외 | Pinterest | Building a dynamic and responsive Pinterest | https://medium.com/pinterest-engineering/building-a-dynamic-and-responsive-pinterest-7d410e99f0a9 | 2026-07-09 | en | Interest Feed, Quick Save | companies/global/pinterest/posts/undated-pinterest-interest-feed.md | Pinterest Engineering 공식 블로그, 원문 미열람 |
| 분석 | 해외 | YouTube | How Does YouTube Count Views? | https://uppbeat.io/blog/youtube/how-does-youtube-count-views , https://www.subsub.io/blog/how-does-youtube-count-views | 2026-07-09 | en | 조회 판정 기준, 단계적 검증, 어뷰징 탐지, 사용자별 제한 | companies/global/youtube/posts/undated-youtube-view-count-algorithm.md | 크리에이터 대상 3자 해설 글, 이번 조사에서 가장 구체적 |
| 제외 | - | - | 검색 사례("실시간 인기 랭킹 트렌딩 알고리즘 이커머스") | - | 2026-07-09 | - | - | - | 검색 결과가 트렌드 마케팅 글 위주라 구체적 구현 사례를 찾지 못함 |
| 제외 | 국내 | 쿠팡 | 타임딜 활용법/실시간 베스트 관련 검색 결과 | - | 2026-07-09 | ko | 타임딜 알림, 실시간 베스트 | - | 소비자 대상 꿀팁 글 위주, 기술 블로그 없음. 타임딜 알림 UX 존재 확인 정도만 참고 |
| 제외 | 해외 | StockX / GOAT | 스니커 리셀 트렌드/마켓 분석 글 | - | 2026-07-09 | en | most wanted 랭킹 | - | 마케팅/시장 리포트 위주, 내부 알고리즘 근거 없음 |

## 상태 값

| 상태 | 의미 |
| --- | --- |
| 후보 | 수집할 가치가 있어 보이나 아직 본문을 확인하지 않음 |
| 분석 | 검색 스니펫 또는 원문을 확인해 회사별 정리나 분석에 반영함 |
| 제외 | 이 주제와 직접 관련이 낮아 제외함 |

## 작성 규칙

- `URL`에는 공개 접근 가능한 원문 링크를 둔다.
- `접근일`은 `YYYY-MM-DD` 형식으로 쓴다.
- `저장 위치`에는 스크랩한 파일 경로를 쓴다.
- 이번 조사는 WebSearch 결과 스니펫을 기준으로 확인했고, 원문 전체를 열람하지 않았다. 스니펫 기준으로만 확인 가능한 내용은 각 스크랩 문서의 `확인 필요`에 남긴다.
