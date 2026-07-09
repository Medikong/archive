# Global Companies

해외 기업/서비스의 찜(위시리스트)/실시간 인기 랭킹/조회수 집계 관련 공개 사례를 정리한다.

## 회사 인덱스

| 회사 | 조사 상태 | 주요 주제 | 시작 문서 |
| --- | --- | --- | --- |
| Reddit | 분석 | Hot 정렬 알고리즘, 로그 스케일 득표, 시간 가중치 | [reddit/README.md](reddit/README.md) |
| Hacker News | 분석 | 거듭제곱(power law) 시간 감쇠 랭킹 | [hackernews/README.md](hackernews/README.md) |
| YouTube | 분석 | 조회 판정 기준, 단계적 검증(freeze), 어뷰징 탐지 | [youtube/README.md](youtube/README.md) |
| Redis (공식) | 분석 | Sorted Set 기반 실시간 리더보드 표준 구현 패턴 | [redis/README.md](redis/README.md) |
| Alibaba Cloud | 분석 | 플래시 세일 시스템의 계층형 트래픽 필터링, Lua 원자적 재고/카운터 처리 | [alibaba-cloud/README.md](alibaba-cloud/README.md) |
| Facebook (Meta) | 분석 | 좋아요 카운터 샤딩(postId 기준), CDC 기반 비동기 집계 | [facebook/README.md](facebook/README.md) |

## 이번 조사에서 얻은 해외 사례 포인트

- [Reddit](https://medium.com/hacking-and-gonzo/how-reddit-ranking-algorithms-work-ef111e33d0d9)과 [Hacker News](https://sangaline.com/post/reverse-engineering-the-hacker-news-ranking-algorithm/)는 실시간 인기 랭킹을 수식 수준까지 공개한 몇 안 되는 사례다. 커머스가 아니지만 "점수 + 시간 감쇠" 문제 구조는 DropMong의 오픈 후 랭킹과 본질적으로 같다.
- [YouTube](https://uppbeat.io/blog/youtube/how-does-youtube-count-views)의 조회수 집계는 이번 조사에서 가장 구체적인 자료였다. "우선 집계 → 사후 검증", "동일 사용자 24시간 5회 제한", "동일 IP 반복 탐지" 같은 패턴은 DropMong의 조회수 dedup 설계를 재검토할 때 참고할 수 있다.
- [Redis 공식 튜토리얼](https://redis.io/tutorials/howtos/leaderboard/)은 Sorted Set으로 실시간 랭킹을 서빙하는 표준 구현(`ZADD`/`ZINCRBY`/`ZREVRANGE`)을 제공한다. 특정 커머스 회사 사례는 아니지만, 점수 계산 공식(Reddit/HN/네이버-다음)을 실제로 "빠르게 조회 가능한 TOP N"으로 만드는 서빙 계층 문제를 메꿔준다.
- [Alibaba Cloud](https://www.alibabacloud.com/blog/high-concurrency-practices-of-redis-snap-up-system_597858)의 플래시 세일 시스템 사례는 DropMong의 "드롭 오픈" 순간과 트래픽 패턴이 가장 유사하다. 계층형 필터링, Lua 원자적 재고 차감, `SET NX EX` 분산 락은 팀의 기존 쿠폰 Redis admission-gate 패턴과도 같은 계열이다.
- [Facebook(Meta)](https://medium.com/@V9vek/5-billion-likes-a-day-the-system-design-behind-facebooks-like-button-501e4bf1044d)의 좋아요 버튼 사례는 `postId` 기준 샤딩과 "즉시 쓰기+비동기 집계" 2단계 구조로, 핫키 카운터 문제(`analysis/06`)에 가장 구체적인 답을 준다.

## 회사별 폴더 규칙

```text
{company}/
  README.md
  posts/
```

- 회사 폴더명은 소문자 kebab-case로 쓴다.
- `README.md`는 회사별 요약과 주요 설계 포인트를 담는다.
- `posts/`에는 포스트별 스크랩을 둔다.
