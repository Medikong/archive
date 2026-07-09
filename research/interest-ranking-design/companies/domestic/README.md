# Domestic Companies

국내 기업의 찜(위시리스트)/실시간 인기 랭킹 관련 공개 사례를 회사별로 정리한다.

## 회사 인덱스

| 회사 | 조사 상태 | 주요 주제 | 시작 문서 |
| --- | --- | --- | --- |
| 무신사 | 분석 | 실시간 베스트 랭킹, 어뷰징/조작 방지, 공정성 | [musinsa/README.md](musinsa/README.md) |
| 29CM | 분석 | 좋아요 API 마이크로서비스 분리, 무중단 마이그레이션 | [29cm/README.md](29cm/README.md) |
| 크림(KREAM) | 분석 | 한정판 리셀 플랫폼의 랭킹 신호(조회+관심+거래), 기간별 랭킹 분리 | [kream/README.md](kream/README.md) |
| 네이버/다음 | 분석 | 실시간 급상승 판정(평균 대비 증가율), 알고리즘 신뢰성 검증 사례 | [naver-daum-trending/README.md](naver-daum-trending/README.md) |

## 이번 조사에서 얻은 국내 사례 포인트

- [무신사](https://www.musinsa.com/content/mz/6622)는 조회, 구매 같은 실시간 활동 지표를 랭킹에 반영하면서도 어뷰징과 조작 없는 공정한 경쟁을 핵심 가치로 제시한다. 신호 집계 방식 자체의 상세 구현은 공개되지 않았다.
- [29CM](https://medium.com/29cm/api-v2-%EC%A0%84%ED%99%98%EA%B3%BC-db-%EB%AC%B4%EC%A4%91%EB%8B%A8-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-%ED%9B%84%EA%B8%B0-8b39eb0db566)는 모놀리식에 있던 좋아요 API를 마이크로서비스로 분리하면서 SQS 기반 이중 기록으로 데이터 동기화를 처리했다. 기능을 나중에 분리할수록 마이그레이션 비용이 커진다는 점을 보여준다.
- [크림(KREAM)](https://kream.co.kr/content/ranking_all-rising)은 DropMong과 가장 가까운 도메인(한정판 거래)이다. 랭킹을 "조회+관심(찜)+거래" 복합 신호로 구성하고, 급상승(최근 3일)과 월간을 분리 제공한다는 점이 DropMong의 오픈 전/오픈 후 랭킹 이원화 설계를 뒷받침한다.
- 세 사례 모두 검색 스니펫/제품 화면 기준 확인이라, 실제 트래픽 규모나 내부 아키텍처 상세는 확인하지 못했다. 결제/쿠폰 도메인(`바운디드-컨텍스트-분석.md`)만큼 상세한 장애 postmortem은 아니다.

## 회사별 폴더 규칙

```text
{company}/
  README.md
  posts/
```

- 회사 폴더명은 소문자 kebab-case로 쓴다.
- `README.md`는 회사별 요약과 주요 설계 포인트를 담는다.
- `posts/`에는 포스트별 스크랩을 둔다.
