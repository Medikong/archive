---
id: REQ.A.07
title: 관심 신호 및 실시간 인기 랭킹 요구사항 정의
type: requirements
status: draft
tags: [requirements, dropmong, interest, wishlist, ranking]
source: local
created: 2026-07-09
updated: 2026-07-09
---

# 관심 신호 및 실시간 인기 랭킹 요구사항 정의

## 기본 정보

- Requirements ID: `REQ.A.07`
- 프로젝트: DropMong
- 작성 기준일: 2026-07-09
- 주요 사용자: 구매자, DropMong 운영자, 브랜드 운영자
- 주요 이해관계자: catalog-service 담당자, order-service 담당자, notification-service 담당자
- 설계 범위: 찜(위시리스트) 등록/해제/조회, 오픈 전 드롭 랭킹 산출, 오픈 후 드롭 랭킹 산출, 드롭 상태 전환 감지, 조회수 중복/매크로 방지
- 제외 범위: 알림 신청(구독) 기능, 비로그인 사용자 조회수 집계, 공유 기반 신호, 실제 알림 발송(notification-service 소관), 검색/개인화 추천(catalog-service 및 향후 search-service 소관)

## 구조 분석

### 문서 역할

- `REQ.A.01`은 드롭 발견부터 구매, 결과 확인까지의 사용자 여정을 다루고, `FR-019`에서 "찜, 알림 신청, 상세 조회, 검색, 구매 시도 로그를 드롭 성과 분석에 사용할 수 있게 저장한다"를 Should 우선순위로 이미 언급했다.
- 이 문서는 `FR-019`가 요구하는 신호 중 **찜과 조회수**를 실제로 소유하고, 이를 바탕으로 오픈 전/오픈 후 드롭의 실시간 인기 랭킹을 산출하는 책임을 정의한다. 알림 신청, 공유, 검색 로그는 이 문서의 책임 밖으로 명시적으로 남긴다.
- `03-service-boundaries.md` 11번 섹션("나중에 분리할 조건")에는 `waiting-room-service`, `search-service`가 이미 후보로 있었고, 이 문서가 다루는 영역도 같은 성격(현재 8개 서비스 어디에도 속하지 않는 신규 바운디드 컨텍스트)이다.
- 마이페이지의 "찜리스트" 메뉴(`UI_A_10_my.md`)와 상품 상세의 "관심 등록" 버튼(`UI_A_02_product_detail.md`)은 이미 UI 설계에 존재하지만, 이를 뒷받침하는 백엔드는 없었다. 이 문서가 그 자리를 채운다.
- 찜(좋아요류 기능)을 모놀리식 서비스 안에 두었다가 나중에 분리하면 무중단 마이그레이션 비용이 크다는 것이 29CM 사례([근거](#레퍼런스))에서 확인된다. 이 문서는 그 교훈을 반영해 **처음부터 독립 서비스로 시작**하는 근거로 삼는다.

### 근거 자료 구조

- `06-event-contracts.md`, `service/contracts/events/dropmong-purchase-events.md`에 이미 정의된 `order.created`, `order.confirmed`, `catalog.drop.updated` 이벤트를 그대로 소비하는 것을 전제로 요구사항을 정의한다.
- `01-domain-model.md`의 드롭 상태 다이어그램(`SCHEDULED → OPEN → SOLD_OUT/CLOSED`)과 재고 버킷 불변조건(`INV-01: reserved_count + confirmed_count <= total_quantity`)을 오픈 후 랭킹 계산의 근거로 삼는다.
- 이 도메인(찜/실시간 랭킹)은 쿠폰·결제 도메인만큼 공개된 장애 postmortem이 풍부하지 않다. 그래서 이 문서의 "도출 근거"는 (1) 실제 기업의 기능 발표/마이그레이션 사례 2건, (2) 조회수 중복방지에 대한 공개된 구현 패턴, (3) 이 대화에서 도출한 내부 설계 분석을 함께 사용한다. `REQ.A.02`(쿠폰)만큼 장애 사례 밀도가 높지 않다는 점을 열어둔다.
- 외부 사례 조사 원본은 `research/interest-ranking-design/`에 별도로 정리했다 (`companies/domestic/musinsa`, `companies/domestic/29cm`, `analysis/01-view-count-dedup.md`). `archive/바운디드-컨텍스트-분석.md`(올리브영/컬리 조사 풀)는 쿠폰/주문/알림/품절 시나리오만 다뤄 이 문서의 주제를 커버하지 않으며, 이번 조사가 찜/랭킹 영역의 첫 조사 풀이다.

## 문제 정의

- 사용자가 겪는 문제: 오픈 전 드롭이 많을 때 어떤 드롭이 기대를 받고 있는지 알 수 없고, 오픈 후에도 재고 규모가 다른 드롭들 사이에서 무엇이 진짜 잘 팔리고 있는지 비교할 방법이 없다.
- 현재 방식의 한계: `catalog-service`는 상품/드롭의 정적 정보만 제공하고, "지금 얼마나 관심을 받고 있는가"라는 동적 신호를 계산하는 주체가 없다.
- 이 프로젝트가 해결해야 하는 것: 찜/조회 신호를 축적해 오픈 전 드롭의 기대치를 랭킹으로 보여주고, 오픈 후에는 재고 규모에 왜곡되지 않는 방식으로 실시간 인기를 계산한다.
- 해결하지 않는 것: 개인화 추천, 검색 relevance 튜닝, 실제 알림 발송 로직(수신 대상 결정과 발송은 notification-service 소관), 매크로/봇의 정교한 탐지(기본적인 중복 집계 방지 수준까지만 다룸).

## 도출 근거

| 근거 | 확인한 신호 | DropMong 요구로 바꾼 내용 |
| --- | --- | --- |
| 29CM "좋아요" API V2 전환 무중단 마이그레이션 | 모놀리식에 있던 좋아요 기능을 마이크로서비스(PostgreSQL→MySQL)로 옮기며 SQS 기반 이중 기록으로 데이터 동기화를 처리했다. 기능을 나중에 분리할수록 마이그레이션 비용과 리스크가 커진다. | 찜 기능은 user-service나 catalog-service에 임시로 얹지 않고, 처음부터 독립 서비스(interest-service)로 시작한다. |
| 무신사 실시간 베스트 랭킹 서비스 | 조회, 구매 등 실시간 활동 지표를 랭킹에 반영하되, 어뷰징과 조작 없이 공정한 경쟁을 유지하는 것을 서비스의 핵심 가치로 제시한다. | 실시간 인기 신호(조회수)는 원시 트래픽을 그대로 반영하지 않고, 중복/어뷰징 방지 로직을 거친 값만 랭킹에 반영한다. |
| 조회수 중복 집계 방지 구현 패턴(Redis `SETNX`/`EX` 기반) | 동일 사용자의 반복 조회를 짧은 TTL 동안 하나의 키로 막아, 새로고침만으로 조회수가 늘어나지 않게 하는 방식이 일반적으로 쓰인다. | 로그인 사용자 기준 `SET NX EX 300 "viewed:{dropId}:{userId}"` 키로 5분 dedup을 적용한다. |
| 내부 설계 분석 (재고 규모 왜곡 문제) | 오픈 후 랭킹을 raw 구매 개수로만 매기면 재고가 적은 한정판이 재고가 많은 드롭에 항상 밀린다. | `sell_through_rate(confirmed_count/total_quantity) / elapsed_minutes` 공식으로 재고 규모를 정규화한다. |
| 내부 설계 분석 (공유 스키마 결합) | `NotificationRequestedEvent.orderId`가 필수 필드라 주문과 무관한 알림(찜한 드롭 오픈 알림)을 지금 스키마로 표현할 수 없다. | 알림 신청/매진임박 알림은 스키마 확장 협의가 끝날 때까지 1차 스코프에서 제외한다. |
| 네이버/다음 실시간 급상승 검색어 신뢰성 논란 + 내부 설계 분석 | 최근 평균 대비 증가율로 판정하는 "급상승"류 랭킹은 순위 변동이 잦고, 결국 신뢰성 논란으로 서비스가 중단된 이력이 있다. 또한 오픈 전 드롭은 급상승 판정에 필요한 신호 밀도(짧은 구간 내 충분한 이벤트 수) 자체가 부족하다. | 오픈 전 랭킹은 속도/감쇠 기반 "급상승" 계산 대신, 매일 00:00에 리셋되는 단순 누적 카운트로 산출한다. |
| 내부 설계 분석 (찜 카운터 대칭성) | 찜은 `FR-001`에 따라 멱등하게 추가/해제할 수 있는데, 해제해도 랭킹 카운터가 줄지 않으면 같은 사용자가 찜/해제를 반복해 오픈 전 랭킹 점수를 인위적으로 부풀릴 수 있다. | 찜 추가는 카운터를 증가시키고 찜 해제는 카운터를 감소시켜, 순 관심도(net interest)만 랭킹에 반영한다. |
| Toss 대규모 트래픽 캐싱 전략 / Alibaba Cloud 플래시 세일 동시성 / Facebook 좋아요 카운터 샤딩 | 드롭 오픈 순간 인기 드롭 하나에 조회/찜 write가 몰리는 "핫키" 문제가 생길 수 있다. Alibaba는 재고만큼 엄격한 Lua 원자적 처리로, Toss는 로컬 집계 후 배치 flush로, Facebook은 postId 샤딩+비동기 집계로 대응한다. 재고 차감과 달리 찜/조회 카운트는 약간의 지연을 허용해도 된다. | 찜 상태(즉시 정확해야 함)와 랭킹 집계용 카운트(지연 허용 가능)를 다른 정합성 수준으로 다루고, 오픈 직후에는 Toss식 로컬 집계 후 배치 반영으로 완충한다. |

## 페인포인트 개선 매핑

| 문제 ID | 페인포인트 | 개선 방향 | 연결 기능 요구사항 | 연결 비기능 요구사항 | 근거 |
| --- | --- | --- | --- | --- | --- |
| `REQ.A.07.PP-001` | 찜 같은 사용자 관심 기능을 다른 서비스에 얹어두면 나중에 분리할 때 무중단 마이그레이션 비용이 커진다. | 처음부터 독립 서비스(interest-service)로 시작하고 user-service/catalog-service 코드에 얹지 않는다. | `REQ.A.07.FR-001`, `REQ.A.07.FR-002` | `REQ.A.07.NFR-002` | [29CM 좋아요 API V2 전환](https://medium.com/29cm/api-v2-%EC%A0%84%ED%99%98%EA%B3%BC-db-%EB%AC%B4%EC%A4%91%EB%8B%A8-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-%ED%9B%84%EA%B8%B0-8b39eb0db566) |
| `REQ.A.07.PP-002` | 실시간 인기 지표를 사용자에게 그대로 노출하면 어뷰징/조작으로 신뢰를 잃을 수 있다. | 로그인 사용자 기준으로만 조회 신호를 집계해 최소한의 진입장벽을 둔다. | `REQ.A.07.FR-007` | `REQ.A.07.NFR-003` | [무신사 실시간 베스트 랭킹 서비스](https://www.musinsa.com/content/mz/6622) |
| `REQ.A.07.PP-003` | 조회수를 원시 트래픽 그대로 집계하면 같은 사용자의 새로고침만으로 쉽게 부풀려진다. | Redis `SET NX EX`로 사용자/드롭 단위 시간 윈도우 dedup을 적용한다. | `REQ.A.07.FR-004` | `REQ.A.07.NFR-001` | [조회수 중복 방지 구현 패턴](https://velog.io/@horang12/%EB%8F%99%EC%9D%BC%ED%95%9C-%EC%9C%A0%EC%A0%80%EC%9D%98-%EC%A1%B0%ED%9A%8C%EC%88%98%EA%B0%80-%EC%A4%91%EB%B3%B5-%EC%A7%91%EA%B3%84-%EB%90%98%EC%A7%80-%EC%95%8A%EA%B2%8C-%ED%95%98%EB%A0%A4%EB%A9%B4) |
| `REQ.A.07.PP-004` | raw 구매 개수 기준 랭킹은 재고 수량이 적은 한정판이 재고가 많은 드롭에 항상 밀리게 만든다. | 소진율을 경과 시간으로 나눈 정규화 점수로 오픈 후 랭킹을 산출한다. | `REQ.A.07.FR-005` | `REQ.A.07.NFR-004` | 내부 설계 분석 (`01-domain-model.md`의 `INV-01` 근거) |
| `REQ.A.07.PP-005` | 공유 이벤트 스키마(`NotificationRequestedEvent.orderId` 필수)가 주문과 무관한 알림을 표현하지 못해 팀 간 조율 없이는 알림 신청 기능을 완성할 수 없다. | 알림 신청/매진임박 알림은 스키마 확장 협의 전까지 스코프에서 제외한다. | - | `REQ.A.07.NFR-005` | 내부 설계 분석 (`packages/contracts/src/contracts/events.py` 코드 확인) |
| `REQ.A.07.PP-006` | 속도/감쇠 기반 "급상승" 랭킹은 순위가 너무 자주 바뀌어 사용자 신뢰를 잃기 쉽고, 오픈 전 드롭은 애초에 급상승 판정에 필요한 신호 밀도가 부족해 비효율적이다. | 오픈 전 랭킹은 감쇠 계산 없이 당일(00:00 리셋) 누적 카운트로 단순화한다. 실제로 폭발적인 관심을 받는 드롭은 별도 감지 로직 없이도 누적 카운트만으로 상위권에 오른다. | `REQ.A.07.FR-003` | `REQ.A.07.NFR-006` | [네이버/다음 실시간 급상승 검색어 알고리즘](https://www.clien.net/service/board/park/5599879) |
| `REQ.A.07.PP-007` | 찜 해제가 랭킹 카운터에 반영되지 않으면, 찜/해제를 반복해 오픈 전 랭킹 점수를 인위적으로 부풀릴 수 있다. | 찜 추가는 카운터 증가, 찜 해제는 카운터 감소로 대칭 처리해 순 관심도만 반영한다. | `REQ.A.07.FR-009` | `REQ.A.07.NFR-007` | 내부 설계 분석 (`FR-001` 찜 멱등 토글 구조 확인) |
| `REQ.A.07.PP-008` | 드롭 오픈 순간 인기 드롭 하나의 찜/조회 카운터(핫키)에 write가 몰리면 Redis 부하가 커지고, 그 여파로 다른 드롭의 랭킹 조회 응답까지 느려질 수 있다. | 찜 상태(즉시 정확)와 랭킹 집계용 카운트(지연 허용)를 분리하고, 오픈 직후에는 애플리케이션 서버 로컬 집계 후 짧은 주기로 배치 반영하는 완충 설계를 1차 방향으로 삼는다. | `REQ.A.07.FR-010` | `REQ.A.07.NFR-008` | [Toss 대규모 트래픽 처리](https://toss.tech/article/27600), [Alibaba Cloud 플래시 세일 동시성](https://www.alibabacloud.com/blog/high-concurrency-practices-of-redis-snap-up-system_597858), [Facebook 좋아요 카운터 샤딩](https://medium.com/@V9vek/5-billion-likes-a-day-the-system-design-behind-facebooks-like-button-501e4bf1044d) |

## 목표

- 사용자 목표: 오픈 전에는 기대되는 드롭을, 오픈 후에는 지금 가장 빠르게 소진되는 드롭을 한눈에 파악한다.
- 비즈니스 목표: `FR-019`가 요구하는 드롭 성과 분석 데이터(찜, 조회)를 실제로 축적한다.
- 운영 목표: 재고 수량이 적은 한정판이 재고가 많은 드롭에 비해 항상 랭킹에서 불리해지지 않도록 한다.
- 기술 목표: 기존 서비스(catalog, order) 코드를 수정하지 않고 이미 발행 중인 이벤트의 신규 consumer로만 연동한다.

## 사용자 유형

| Actor ID | 사용자 | 목표 | 주요 행동 |
| --- | --- | --- | --- |
| `ACTOR-BUYER` | 구매자 | 관심 있는 드롭을 저장하고, 기대되는/인기 있는 드롭을 빠르게 찾는다 | 찜 추가/해제, 찜 목록 조회, 드롭 상세 조회, 랭킹 목록 조회 |
| `ACTOR-DROPMONG-OPERATOR` | DropMong 운영자 | 드롭별 관심도와 판매 속도를 근거로 운영 판단을 한다 | 랭킹/찜 통계 조회 |
| `ACTOR-BRAND-OPERATOR` | 브랜드 운영자 | 자사 드롭의 사전 반응과 판매 속도를 확인한다 | 드롭별 찜/조회/랭킹 통계 조회 |

## 기능 요구사항

| Req ID | 요구사항 | 사용자 | 우선순위 | 연결 Page/UC |
| --- | --- | --- | --- | --- |
| `REQ.A.07.FR-001` | 사용자는 로그인 후 드롭을 찜 목록에 추가하거나 해제한다 | 구매자 | Must | [PAGE.A.02](../10-sitemap/PAGE_A_02_product_detail.md), [PAGE.A.10](../10-sitemap/PAGE_A_10_my.md) |
| `REQ.A.07.FR-002` | 사용자는 자신의 찜 목록을 조회한다 | 구매자 | Must | [PAGE.A.10](../10-sitemap/PAGE_A_10_my.md) |
| `REQ.A.07.FR-003` | 시스템은 `SCHEDULED` 상태 드롭에 대해 당일(00:00 기준) 누적된 찜/조회수 가중합으로 오픈 전 랭킹을 산출하며, 매일 자정 카운터를 리셋한다 | 구매자, DropMong 운영자 | Must | [PAGE.A.01](../10-sitemap/PAGE_A_01_homepage.md) |
| `REQ.A.07.FR-004` | 시스템은 로그인 사용자의 드롭 조회를 5분 단위로 중복 집계하지 않는다 | - | Must | - |
| `REQ.A.07.FR-005` | 시스템은 `OPEN` 상태 드롭에 대해 `sell_through_rate / elapsed_minutes` 공식으로 오픈 후 랭킹을 산출한다 | 구매자, DropMong 운영자 | Must | [PAGE.A.01](../10-sitemap/PAGE_A_01_homepage.md) |
| `REQ.A.07.FR-006` | 시스템은 `catalog.drop.updated` 이벤트를 구독해 드롭 상태 전환(`SCHEDULED → OPEN`)에 따라 랭킹 리스트를 전환한다 | - | Must | - |
| `REQ.A.07.FR-007` | 시스템은 비로그인 사용자의 조회는 랭킹 집계에서 제외한다 | - | Must | - |
| `REQ.A.07.FR-008` | 시스템은 드롭별 찜 카운트를 조회 가능한 형태로 제공한다 | DropMong 운영자, 브랜드 운영자 | Should | TBD |
| `REQ.A.07.FR-009` | 시스템은 찜 해제 시 오픈 전 랭킹 카운터를 즉시 감소시킨다 | - | Must | - |
| `REQ.A.07.FR-010` | 시스템은 찜 상태(사용자에게 즉시 반영되는 값)와 랭킹 집계용 카운트(짧은 지연을 허용하는 값)를 분리해서 처리한다 | - | Should | - |

## 비기능 요구사항

| Req ID | 요구사항 | 기준 | 연결 문서 |
| --- | --- | --- | --- |
| `REQ.A.07.NFR-001` | 조회수 중복 집계 방지는 사용자/드롭 단위 시간 윈도우로 처리한다 | 같은 `(userId, dropId)` 조합은 5분 안에 최대 1회만 조회수에 반영된다 | TBD |
| `REQ.A.07.NFR-002` | 기존 catalog-service, order-service 코드는 수정하지 않고 이벤트 consumer로만 연동한다 | 배포/코드리뷰 시 확인 | `03-service-boundaries.md` |
| `REQ.A.07.NFR-003` | 조회수 중복 방지는 완전한 봇 탐지가 아니라 로그인 요구 + 시간 윈도우 dedup 수준으로 한정한다 | 설계 리뷰 | - |
| `REQ.A.07.NFR-004` | 오픈 후 랭킹은 재고 수량 차이로 왜곡되지 않아야 한다 | 서로 다른 `total_quantity`를 가진 두 드롭이 동일한 소진 속도를 보이면 랭킹 점수가 비슷하다 | `01-domain-model.md` |
| `REQ.A.07.NFR-005` | 공유 이벤트 스키마 변경이 필요한 기능은 별도 협의 없이 릴리즈하지 않는다 | `NotificationRequestedEvent` 등 `packages/contracts` 변경은 사전 협의 이슈로 기록된다 | - |
| `REQ.A.07.NFR-006` | 오픈 전 랭킹은 실시간 감쇠/속도 계산 없이 당일 누적 카운트만 사용한다 | 오픈 전 랭킹 점수 계산 로직에 시간 가중치/감쇠 항이 없다 | `analysis/02-realtime-ranking-score-design.md` |
| `REQ.A.07.NFR-007` | 찜 카운터는 찜 추가/해제에 대해 대칭적으로 증감한다 | 동일 사용자가 짧은 시간 안에 찜→해제→찜을 반복해도, 최종 찜 상태와 랭킹 카운터에 반영된 순증감이 일치한다 | - |
| `REQ.A.07.NFR-008` | 특정 드롭 하나에 조회/찜 write가 몰려도(핫키), 다른 드롭의 랭킹 조회 응답 시간에 영향을 주지 않는다 | 단일 dropId에 대한 카운터 write 폭주 상황에서도 랭킹 API 응답 시간이 정상 범위를 유지한다 (구체적 임계치는 부하테스트 후 확정) | `analysis/06-hotkey-counter-concurrency.md` |

## 제약 조건

- 정책/법적 제약: 없음 (찜/조회 데이터는 개인정보가 아닌 행동 로그 수준으로 취급)
- 기술 제약: `catalog.drop.updated` 이벤트는 `06-event-contracts.md` 문서에는 정의돼 있으나 `packages/contracts` 코드에는 아직 없어 추가가 필요하다. `NotificationRequestedEvent`의 `orderId` 필수 필드 때문에 향후 매진임박 알림 연동 시 공유 스키마 확장이 필요하다(1차 스코프 제외 사유).
- 일정 제약: 1차 스코프는 찜 + 로그인 사용자 조회수만 다루고, 알림 신청/공유/비로그인 조회는 향후 확장으로 미룬다.
- 데이터 제약: 오픈 후 랭킹은 `order-service`가 소유한 `total_quantity`, `confirmed_count` 필드에 의존한다. 이 서비스는 해당 값의 authoritative owner가 아니다.
- 운영 제약: coupon-service의 `coupon.issued` 이벤트는 아직 정식 topic 계약이 없어 이 문서의 랭킹 신호에는 포함하지 않는다.

## 수용 기준

- 로그인 사용자가 찜을 추가/해제하면 해당 드롭의 찜 카운트가 즉시 반영된다.
- 같은 로그인 사용자가 5분 이내 같은 드롭을 여러 번 조회해도 조회수가 한 번만 증가한다.
- 드롭 상태가 `SCHEDULED`에서 `OPEN`으로 바뀌면 해당 드롭이 `ranking:upcoming`에서 빠지고 `ranking:hot` 계산 대상으로 전환된다.
- 오픈 전 랭킹 카운터는 매일 00:00에 리셋되며, 리셋 이후에는 그날의 찜/조회수만 랭킹에 반영된다.
- 사용자가 찜을 해제하면 해당 드롭의 오픈 전 랭킹 카운터가 즉시 감소한다. 같은 사용자가 짧은 시간 안에 찜/해제를 반복해도 최종 카운터 값은 실제 찜 상태와 일치한다.
- 재고 수량이 서로 다른 두 드롭이 동일한 소진 속도를 보이면 오픈 후 랭킹에서 비슷한 순위를 받는다.
- 비로그인 사용자의 조회는 랭킹 점수에 반영되지 않는다.
- 모든 요구사항 ID는 `REQ.A.07.FR-*` 또는 `REQ.A.07.NFR-*` 형식으로 기록된다.

## 연관 태그

🏷️ 플로우 참조: FLOW.A.XX | 페이지 참조: [PAGE.A.01](../10-sitemap/PAGE_A_01_homepage.md), [PAGE.A.02](../10-sitemap/PAGE_A_02_product_detail.md), [PAGE.A.10](../10-sitemap/PAGE_A_10_my.md) | UI 참조: [UI.A.02](../20-ui/UI_A_02_product_detail.md), [UI.A.10](../20-ui/UI_A_10_my.md) | UC 참조: TBD | 영속성 참조: TBD | 서비스 참조: TBD | 시나리오 참조: TBD | API 참조: TBD

## 열린 질문

- coupon-service의 `coupon.issued` 이벤트 payload와 topic 이름을 coupon 담당 팀원과 언제 확정할 것인가.
- 매진임박 알림을 위해 `NotificationRequestedEvent`의 `orderId`를 optional로 바꿀지, `dropId` 필드를 추가할지 공유 패키지 담당자와 논의 필요.
- 오픈 후 랭킹 공식의 최소 elapsed 바닥값(현재 1분)과 조회수 dedup 윈도우(현재 5분)가 실제 부하테스트 후에도 적정한지.
- 무신사 사례처럼 "N명이 보는 중"류의 실시간 표시까지 UI에 노출할 것인가, 아니면 랭킹 목록 정렬까지만 다룰 것인가.
- 핫키 대응(`FR-010`, `NFR-008`)의 구체적 구현(로컬 버퍼링 주기, 배치 크기, 어느 정도 트래픽부터 적용할지)은 실제 예상 트래픽 규모가 나온 뒤 결정한다. 1차 스코프에서는 `ZINCRBY` 단일 키만으로 충분할 수도 있다.

## 확인 필요

- 찜 카운트를 드롭 단위 실시간 통계로 운영자에게 노출할 API 형태(`FR-008`)는 60-service/70-api 단계에서 구체화 필요.
- 언어/스택: Python(FastAPI) 채택 근거 — `packages/contracts`, `packages/kafka-utils`에 이미 order/payment/notification이 쓰는 이벤트 스키마와 Kafka 헬퍼가 있어 재사용 가능. Go 트랙(`go-contracts`, `go-platform`)에는 Kafka 관련 코드가 없음.

## 새로 드러난 가장자리

- 이 자료가 답한 질문: 찜과 실시간 랭킹은 UI/요구사항 문서에는 있었지만 실제로 소유하는 서비스가 없었다는 공백을 확인했고, 처음부터 독립 서비스로 분리하는 근거(29CM 사례)와 조회수 신뢰성 확보 근거(무신사 사례, Redis dedup 패턴)를 찾았다.
- 아직 남은 질문: 알림 신청과 매진임박 알림을 언제 붙일지, coupon-service 이벤트를 언제 반영할지는 다른 팀원과의 일정 조율에 달려있다.
- 검증되지 않은 전제: 로그인 사용자만 조회수를 집계해도 오픈 전 드롭의 관심도를 충분히 대표할 수 있다고 가정했다. 비로그인 트래픽 비중이 크면 이 가정을 재검토해야 한다.
- 약한 연결: 이 도메인은 결제/쿠폰 도메인만큼 공개된 장애 사례가 많지 않아, 도출 근거의 다양성이 `REQ.A.02`보다 낮다. 실제 부하테스트 결과가 나오면 도출 근거를 보강해야 한다.
- 다음에 확인할 것: `catalog.drop.updated` 이벤트를 `packages/contracts`에 추가하는 작업, coupon/notification 담당자와의 이벤트 계약 협의, dedup 윈도우와 오픈 후 공식의 실측 검증.

## 레퍼런스

- `REQ.A.07.PP-001` 찜 기능의 뒤늦은 분리 비용: 모놀리식에 있던 "좋아요" API를 마이크로서비스로 옮기며 무중단 마이그레이션을 위해 SQS 기반 이중 기록 전략을 썼다. DropMong은 찜 기능을 처음부터 독립 서비스로 시작해 이 비용을 피한다. 근거: [29CM API V2 전환과 DB 무중단 마이그레이션 후기](https://medium.com/29cm/api-v2-%EC%A0%84%ED%99%98%EA%B3%BC-db-%EB%AC%B4%EC%A4%91%EB%8B%A8-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-%ED%9B%84%EA%B8%B0-8b39eb0db566)
- `REQ.A.07.PP-002` 실시간 인기 지표의 공정성: 무신사는 실시간 베스트 랭킹을 어뷰징/조작 없는 공정한 경쟁으로 제시한다. DropMong도 로그인 요구를 최소 방어선으로 삼아 조회 신호의 신뢰성을 확보한다. 근거: [무신사 스토어, 국내 최초 실시간 베스트 랭킹 서비스 제공](https://www.musinsa.com/content/mz/6622)
- `REQ.A.07.PP-003` 조회수 중복 집계: 동일 사용자의 반복 조회를 짧은 TTL 동안 하나의 키로 막는 Redis `SETNX`/`EX` 패턴이 조회수 어뷰징 방지에 일반적으로 쓰인다. 근거: [동일한 유저의 조회수가 중복 집계 되지 않게 하려면?](https://velog.io/@horang12/%EB%8F%99%EC%9D%BC%ED%95%9C-%EC%9C%A0%EC%A0%80%EC%9D%98-%EC%A1%B0%ED%9A%8C%EC%88%98%EA%B0%80-%EC%A4%91%EB%B3%B5-%EC%A7%91%EA%B3%84-%EB%90%98%EC%A7%80-%EC%95%8A%EA%B2%8C-%ED%95%98%EB%A0%A4%EB%A9%B4)
- `REQ.A.07.PP-006` 급상승(속도/감쇠 기반) 랭킹의 신뢰성 문제: 최근 평균 대비 증가율로 급상승을 판정하는 방식은 순위 변동이 잦아 신뢰성 논란을 일으킬 수 있다는 것이 네이버/다음 실시간 급상승 검색어 서비스 사례에서 확인된다. DropMong은 오픈 전 랭킹을 감쇠 계산 없는 당일 누적 카운트로 단순화해 이 문제를 피한다. 근거: [네이버/다음 실시간 급상승 검색어 알고리즘 설명](https://www.clien.net/service/board/park/5599879)
- `REQ.A.07.PP-008` 드롭 오픈 순간 핫키 문제: Alibaba Cloud는 재고 차감처럼 엄격한 정확성이 필요한 경우 Lua 원자적 처리로, Toss는 정확성보다 처리량이 중요한 경우 로컬 집계 후 배치 flush로, Facebook은 postId 샤딩+비동기 집계로 대응한다. 찜/조회 카운트는 재고만큼 엄격할 필요가 없어 Toss식 완충 설계를 1차 방향으로 삼는다. 근거: [Toss 서버 증설 없이 처리하는 대규모 트래픽](https://toss.tech/article/27600), [Alibaba Cloud High-Concurrency Practices of Redis: Snap-Up System](https://www.alibabacloud.com/blog/high-concurrency-practices-of-redis-snap-up-system_597858), [Facebook 5 Billion Likes a Day](https://medium.com/@V9vek/5-billion-likes-a-day-the-system-design-behind-facebooks-like-button-501e4bf1044d)

- 29CM: [API V2 전환과 DB 무중단 마이그레이션 후기](https://medium.com/29cm/api-v2-%EC%A0%84%ED%99%98%EA%B3%BC-db-%EB%AC%B4%EC%A4%91%EB%8B%A8-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-%ED%9B%84%EA%B8%B0-8b39eb0db566)
- 무신사: [무신사 스토어, 국내 최초 실시간 베스트 랭킹 서비스 제공](https://www.musinsa.com/content/mz/6622)
- 조회수 중복 방지: [동일한 유저의 조회수가 중복 집계 되지 않게 하려면?](https://velog.io/@horang12/%EB%8F%99%EC%9D%BC%ED%95%9C-%EC%9C%A0%EC%A0%80%EC%9D%98-%EC%A1%B0%ED%9A%8C%EC%88%98%EA%B0%80-%EC%A4%91%EB%B3%B5-%EC%A7%91%EA%B3%84-%EB%90%98%EC%A7%80-%EC%95%8A%EA%B2%8C-%ED%95%98%EB%A0%A4%EB%A9%B4)
- 네이버/다음: [실시간 급상승 검색어 알고리즘 설명](https://www.clien.net/service/board/park/5599879)
- Toss: [서버 증설 없이 처리하는 대규모 트래픽](https://toss.tech/article/27600)
- Alibaba Cloud: [High-Concurrency Practices of Redis: Snap-Up System](https://www.alibabacloud.com/blog/high-concurrency-practices-of-redis-snap-up-system_597858)
- Facebook (Meta): [5 Billion Likes a Day: The System Design Behind Facebook's Like Button](https://medium.com/@V9vek/5-billion-likes-a-day-the-system-design-behind-facebooks-like-button-501e4bf1044d)
