# Pinterest

## 개요

- 구분: 해외 비주얼 큐레이션/발견 플랫폼
- 주요 서비스: Pinterest 앱, "Quick Save"(찜류 기능)
- 조사 초점: 콘텐츠 피드/저장(save) 기능을 지원하는 아키텍처, 공식 엔지니어링 블로그 존재 여부

## 핵심 설계 포인트

| 주제 | 확인한 내용 | 출처 |
| --- | --- | --- |
| 엔지니어링 블로그 존재 | Pinterest Engineering이 공식 Medium 블로그를 운영하며, Following Feed/Interest Feed/Picked For You 아키텍처를 다룬 글이 있음 | [Building a dynamic and responsive Pinterest](https://medium.com/pinterest-engineering/building-a-dynamic-and-responsive-pinterest-7d410e99f0a9) |
| 저장(찜류) 기능 | "Quick Save" 기능이 있어 콘텐츠를 자동으로 위시리스트에 담을 수 있다고 설명됨 | 검색 결과 요약 |
| 대규모 데이터 인프라 | AWS와 협업해 로깅/모니터링, 데이터 접근 관리를 구성한 사례가 별도로 존재 (AWS 블로그 시리즈) | [Inside Pinterest's Custom Spark Job logging](https://aws.amazon.com/blogs/containers/inside-pinterests-custom-spark-job-logging-and-monitoring-on-amazon-eks-using-aws-for-fluent-bit-amazon-s3-and-adot/) |

## 참고할 점

- Pinterest가 "Interest Feed"라는 이름으로 관심 기반 피드 아키텍처를 공식적으로 다룬다는 것 자체가, "관심 신호를 모아 노출 순서를 만든다"는 우리 도메인과 개념적으로 맞닿아 있다.
- 대형 플랫폼도 찜류 기능(Quick Save)을 상품 발견 경험의 핵심 축으로 취급한다는 점을 참고할 수 있다.

## DropMong에 그대로 가져오면 안 되는 점

- 이번 검색으로는 "Interest Feed"나 "Quick Save"의 구체적인 저장/집계 구현(DB 스키마, 실시간 처리 방식)을 확인하지 못했다. Pinterest는 추천 피드 랭킹이 핵심인 서비스라 DropMong의 "오픈 전/오픈 후 드롭 랭킹"과는 문제 규모와 목적이 다르다.

## 확인 필요

- Pinterest Engineering 블로그의 "Building a dynamic and responsive Pinterest" 원문을 직접 열람해 Interest Feed의 신호 구성 방식을 확인해야 한다.

## 관련 문서

- `posts/undated-pinterest-interest-feed.md`
