# Mercari

## 개요

- 구분: 해외(일본) C2C 중고거래 마켓플레이스 (REQ.A.01 경쟁 비교에도 등장)
- 주요 서비스: Mercari 앱
- 조사 초점: 무중단 마이그레이션과 강한 데이터 정합성 확보 방식

## 핵심 설계 포인트

| 주제 | 확인한 내용 | 출처 |
| --- | --- | --- |
| 마이그레이션 목표 | Zero Downtime Migration을 강한 데이터 정합성(Strong Data Consistency)과 함께 달성하는 설계를 다룸 (Part II 존재로 미루어 연재물로 추정) | [Mercari Engineering: Zero Downtime Migration Part II](https://engineering.mercari.com/en/blog/entry/20241113-designing-a-zero-downtime-migration-solution-with-strong-data-consistency-part-ii/) |
| 구체 기법 | 검색 스니펫만으로는 확인 불가 | - |

## 참고할 점

- Mercari는 공식 엔지니어링 블로그(engineering.mercari.com)를 운영하며 무중단 마이그레이션을 진지하게 다룬다. 이는 29CM 사례("좋아요 API 분리 시 SQS 이중 기록")와 같은 문제의식을 대형 마켓플레이스도 공유한다는 근거로 쓸 수 있다.
- Mercari는 `REQ.A.01`(전체 요구사항 문서)에서 이미 비교 대상 기업으로 언급된 곳이라, 이 조사가 기존 문서와 연결된다.

## DropMong에 그대로 가져오면 안 되는 점

- 원문 전체를 확인하지 못해 구체적인 기법(이중 기록, 스키마 전환 순서 등)을 DropMong에 바로 적용하기는 이르다. Part I, II 원문을 직접 열람한 뒤 재평가해야 한다.

## 확인 필요

- 원문 전체 열람 필요. 이 마이그레이션이 찜/좋아요류 기능과 직접 관련된 것인지, 아니면 다른 도메인(예: 상품/주문)의 마이그레이션인지 스니펫만으로는 확인하지 못했다.

## 관련 문서

- `posts/undated-mercari-zero-downtime-migration.md`
