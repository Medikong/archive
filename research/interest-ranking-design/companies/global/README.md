# Global Companies

해외 기업/서비스의 찜(위시리스트)/실시간 인기 랭킹/조회수 집계 관련 공개 사례를 정리한다.

## 회사 인덱스

| 회사 | 조사 상태 | 주요 주제 | 시작 문서 |
| --- | --- | --- | --- |
| Reddit | 분석 | Hot 정렬 알고리즘, 로그 스케일 득표, 시간 가중치 | [reddit/README.md](reddit/README.md) |
| Hacker News | 분석 | 거듭제곱(power law) 시간 감쇠 랭킹 | [hackernews/README.md](hackernews/README.md) |
| Mercari | 후보 | 무중단 마이그레이션, 강한 데이터 정합성 (원문 미열람) | [mercari/README.md](mercari/README.md) |
| Nike SNKRS | 분석 | 드롭 방식(FLOW/LEO/DAN), 관심 신호 기반 응모 심사 | [nike-snkrs/README.md](nike-snkrs/README.md) |
| Pinterest | 후보 | Interest Feed 아키텍처, Quick Save (원문 미열람) | [pinterest/README.md](pinterest/README.md) |
| YouTube | 분석 | 조회 판정 기준, 단계적 검증(freeze), 어뷰징 탐지 | [youtube/README.md](youtube/README.md) |

## 이번 조사에서 얻은 해외 사례 포인트

- [Reddit](https://medium.com/hacking-and-gonzo/how-reddit-ranking-algorithms-work-ef111e33d0d9)과 [Hacker News](https://sangaline.com/post/reverse-engineering-the-hacker-news-ranking-algorithm/)는 실시간 인기 랭킹을 수식 수준까지 공개한 몇 안 되는 사례다. 커머스가 아니지만 "점수 + 시간 감쇠" 문제 구조는 DropMong의 오픈 후 랭킹과 본질적으로 같다.
- [YouTube](https://uppbeat.io/blog/youtube/how-does-youtube-count-views)의 조회수 집계는 이번 조사에서 가장 구체적인 자료였다. "우선 집계 → 사후 검증", "동일 사용자 24시간 5회 제한", "동일 IP 반복 탐지" 같은 패턴은 DropMong의 조회수 dedup 설계를 재검토할 때 참고할 수 있다.
- [Nike SNKRS](https://stashed-sneakers.com/blogs/blog-posts/nike-snkrs-drop-methods-guide-stashed)는 DropMong과 도메인이 사실상 같은(한정판 드롭) 서비스지만, 대부분 서드파티 해설 글이라 내부 알고리즘은 비공개로 확인하지 못했다.
- Mercari와 Pinterest는 공식 엔지니어링 블로그의 **존재**만 확인했고 원문은 아직 열람하지 못했다. 상태를 "후보"로 남기고, 다음 조사에서 원문을 직접 읽어야 한다.

## 회사별 폴더 규칙

```text
{company}/
  README.md
  posts/
```

- 회사 폴더명은 소문자 kebab-case로 쓴다.
- `README.md`는 회사별 요약과 주요 설계 포인트를 담는다.
- `posts/`에는 포스트별 스크랩을 둔다.
