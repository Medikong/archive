---
id: REQ.A.01
title: 한정 상품 드롭 커머스 요구사항 정의
type: requirements
status: draft
tags: [requirements, dropmong, limited-commerce, ecommerce, drop-commerce]
source: local
created: 2026-07-06
updated: 2026-07-07
---

# 한정 상품 드롭 커머스 요구사항 정의

## 기본 정보

- Requirements ID: `REQ.A.01`
- 프로젝트: DropMong
- 작성 기준일: 2026-07-06
- 주요 사용자: 구매자, 브랜드 운영자, DropMong 운영자
- 주요 이해관계자: CS, 온콜 담당자, 결제/배송/알림 연동 담당자, 마케팅 담당자
- 설계 범위: 드롭 발견, 오픈 대기, 비로그인 탐색, 참여 자격 확인, 구매 시도, 재고 배정, 주문/결제 생성, 알림, 운영 모니터링
- 제외 범위: 장기 물류 최적화, 정산 회계, 글로벌 다통화, 중고거래, 광고 수익화, 오프라인 매장 POS

## 구조 분석

### 템플릿 구조

- 요구사항 문서는 `기본 정보 -> 문제 정의 -> 목표 -> 사용자 유형 -> 기능 요구사항 -> 비기능 요구사항 -> 제약 조건 -> 수용 기준 -> 열린 질문` 순서로 작성한다.
- `기능 요구사항`은 사용자가 직접 수행하거나 운영자가 제어해야 하는 행동을 정의한다.
- `비기능 요구사항`은 피크 트래픽, 정합성, 공정성, 관측성, 보안처럼 기능이 실패하지 않게 만드는 기준을 정의한다.
- 이후 sitemap, use case, service, API, persistence 문서가 생기면 각 요구사항의 `연결 Page/UC`와 `연결 문서`를 갱신한다.

### 참고 자료 구조

- `work/tech-blog-library`는 기업 테크 블로그 수집본을 `site-profiles/{site}`와 `posts/{site}/{year}`로 보관한다.
- 이번 문서는 수집본 중 Kurly, Oliveyoung, Bucketplace, Mercari, ZOZO의 커머스 운영 신호를 우선 사용한다.
- 공개 글 분석은 내부 장애 이력 전체가 아니라, 공개된 기술 글에서 반복적으로 확인되는 문제와 개선 방향만 근거로 삼는다.

## 문제 정의

- 사용자가 겪는 문제: 드롭 오픈 시간에는 많은 사용자가 동시에 몰리며, 화면은 성공처럼 보여도 실제 재고 배정, 쿠폰 발급, 주문 생성, 결제 승인 중 하나가 뒤늦게 실패할 수 있다.
- 현재 방식의 한계: 일반 쇼핑몰의 장바구니/결제 흐름은 지속 판매를 전제로 하므로, 정해진 시각에 모두가 같은 조건으로 참여하는 드롭 이벤트의 공정성, 대기 경험, 순간 트래픽, 재고 선점 경쟁을 충분히 설명하지 못한다.
- 이 프로젝트가 해결해야 하는 것: DropMong은 한정 상품의 오픈 전 기대감, 오픈 순간의 공정한 참여, 구매 성공/실패 결과의 신뢰성, 피크 상황에서도 설명 가능한 운영을 제공해야 한다.
- 해결하지 않는 것: 모든 사용자의 구매 성공을 보장하지 않는다. 외부 PG, 택배사, 브랜드 공급망 장애 자체를 제거하지 않는다. 대신 장애가 사용자 경험과 데이터 정합성으로 번지지 않도록 격리하고 설명한다.

## 도출 근거

| 플랫폼 | 버티컬/성격 | 확인된 문제 | DropMong에 반영할 요구 |
| --- | --- | --- | --- |
| Kurly | 식품/새벽배송 커머스 | 장바구니, 가격, 할인, 재고, 배송 권역이 주문 직전까지 변할 수 있다. 카트 데드락, 피크 시간 캐시 의존, 운영 중 접근 차단 필요가 반복된다. | 주문 전 가격/재고/혜택을 서버 기준으로 재검증하고, 드롭 오픈 중에는 캐시 장애와 롤백 불가 상태를 전제로 설계한다. |
| Oliveyoung | 뷰티/온오프라인 커머스 | 세일 피크에 주문, 쿠폰, 인증, MQ, 배송 배정이 동시에 병목이 된다. API 성공 응답과 최종 쿠폰 발급 결과가 어긋나는 문제가 공개 사례로 나타난다. | 사용자에게 보이는 성공과 최종 처리 성공을 분리해 추적하고, 선착순 처리에는 원자적 발급과 재처리 기준이 필요하다. |
| Bucketplace | 홈/콘텐츠 커머스 | 검색, 추천, 광고, 실험 지표, 피처 관리가 커지며 파편화된다. 실험/메트릭 입력 실수와 검색 품질 저하가 운영 리스크가 된다. | 드롭 목록, 추천, 검색, 알림 실험은 동일한 지표 정의와 관측 기준을 공유해야 한다. |
| Mercari | 마켓플레이스/글로벌 커머스 | DNS, 이미지 pull, 대규모 DB 변경, 글로벌 identity/order/payment 확장에서 장애와 운영 비용이 드러난다. 점진 이전, self-service, rollback 가능성이 핵심이다. | 드롭/주문/결제/알림 변경은 한 번에 교체하지 않고 feature flag, shadow mode, 재처리, rollback 기준을 둔다. |
| ZOZO | 패션 커머스 | 카트/결제 시스템 교체, 검색 로그 분석, 즐겨찾기 대규모 무중단 이전, 푸시 토큰 관리처럼 패션 커머스의 탐색/관심/구매 전환 데이터가 크다. | 찜, 알림 신청, 검색 노출, 드롭 참여 로그는 구매 전환과 운영 판단의 핵심 데이터로 보존한다. |

## 페인포인트 개선 매핑

| 문제 ID | 페인포인트 | 개선 방향 | 연결 기능 요구사항 | 연결 비기능 요구사항 | 근거 |
| --- | --- | --- | --- | --- | --- |
| `REQ.A.01.PP-001` | 오픈 순간 주문/재고 트랜잭션 병목 | 주문 생성, 재고 배정, 결제 요청을 하나의 긴 트랜잭션으로 묶지 않고 책임과 커밋 경계를 분리한다. | `REQ.A.01.FR-008`, `REQ.A.01.FR-009`, `REQ.A.01.FR-011` | `REQ.A.01.NFR-002`, `REQ.A.01.NFR-005`, `REQ.A.01.NFR-013` | [Oliveyoung 주문·결제 트랜잭션](https://oliveyoung.tech/2022-12-13/oliveyoung-transaction-orderstock/) |
| `REQ.A.01.PP-002` | 품절 정보 조회 지연 | 품절/판매 가능 상태를 DB 직접 조회에만 의존하지 않고 이벤트 기반 조회 모델로 분리한다. | `REQ.A.01.FR-001`, `REQ.A.01.FR-002`, `REQ.A.01.FR-012`, `REQ.A.01.FR-015` | `REQ.A.01.NFR-005`, `REQ.A.01.NFR-006`, `REQ.A.01.NFR-012` | [Oliveyoung 품절 시스템 현대화](https://oliveyoung.tech/2025-12-15/kafka-streams-for-out-of-stock/) |
| `REQ.A.01.PP-003` | 재고 캐시 장애 시 대기 시간 증가 | 재고 조회 캐시 장애 시 무의미한 retry 대기를 줄이고, circuit breaker와 fallback 기준을 둔다. | `REQ.A.01.FR-002`, `REQ.A.01.FR-012`, `REQ.A.01.FR-015` | `REQ.A.01.NFR-006`, `REQ.A.01.NFR-007`, `REQ.A.01.NFR-018` | [Oliveyoung CircuitBreaker](https://oliveyoung.tech/2023-08-31/circuitbreaker-inventory-squad/) |
| `REQ.A.01.PP-004` | 캐시 의존 구조의 장애 전파 | shared cache가 장애를 일으켜도 핵심 API가 한꺼번에 하위 서비스로 몰리지 않도록 캐시 경계와 대체 경로를 분리한다. | `REQ.A.01.FR-015`, `REQ.A.01.FR-016`, `REQ.A.01.FR-017` | `REQ.A.01.NFR-005`, `REQ.A.01.NFR-007`, `REQ.A.01.NFR-018` | [Kurly OMS MSA Architecture](https://helloworld.kurly.com/blog/oms-msa-architecture-1) |
| `REQ.A.01.PP-005` | 카트/체크아웃 동시성 데드락 | 동일 사용자 다중 세션과 중복 구매 시도를 고려해 사용자, 상품, 드롭 단위 중복 제어와 idempotency를 적용한다. | `REQ.A.01.FR-007`, `REQ.A.01.FR-010`, `REQ.A.01.FR-011` | `REQ.A.01.NFR-002`, `REQ.A.01.NFR-003`, `REQ.A.01.NFR-011` | [Kurly 카트 개발 연대기](https://helloworld.kurly.com/blog/my-cart-development-history) |
| `REQ.A.01.PP-006` | 구매 경로 롤백 어려움 | 드롭 구매 핵심 경로 변경은 feature flag, shadow mode, 단계적 전환, rollback 기준을 먼저 정의한 뒤 적용한다. | `REQ.A.01.FR-016`, `REQ.A.01.FR-017`, `REQ.A.01.FR-018` | `REQ.A.01.NFR-014`, `REQ.A.01.NFR-017`, `REQ.A.01.NFR-018` | [Kurly 카트 개발 연대기](https://helloworld.kurly.com/blog/my-cart-development-history) |
| `REQ.A.01.PP-007` | 운영 차단 경로 부재 | 피크 중 특정 드롭, 화면, API, 사용자 그룹을 배포 없이 차단하고 사용자에게 점검/혼잡 안내를 제공한다. | `REQ.A.01.FR-016`, `REQ.A.01.FR-018` | `REQ.A.01.NFR-008`, `REQ.A.01.NFR-016`, `REQ.A.01.NFR-018` | [Kurly AccessBlock](https://helloworld.kurly.com/blog/access-block-1) |
| `REQ.A.01.PP-008` | 비동기 재고 반영 실패와 보상 부재 | 재고 회수, 주문 반영, 알림 발송 실패는 재처리 큐와 운영 타임라인에 남기고 보상 처리를 가능하게 한다. | `REQ.A.01.FR-013`, `REQ.A.01.FR-017`, `REQ.A.01.FR-018` | `REQ.A.01.NFR-004`, `REQ.A.01.NFR-009`, `REQ.A.01.NFR-012` | [Kurly AccessBlock](https://helloworld.kurly.com/blog/access-block-1) |
| `REQ.A.01.PP-009` | 사용자 성공 응답과 최종 처리 결과 불일치 | API 접수 성공, 재고 배정 성공, 주문 생성 성공, 결제 승인 성공을 별도 상태로 추적하고 사용자/CS가 확인할 수 있게 한다. | `REQ.A.01.FR-011`, `REQ.A.01.FR-012`, `REQ.A.01.FR-018` | `REQ.A.01.NFR-003`, `REQ.A.01.NFR-004`, `REQ.A.01.NFR-012` | [Oliveyoung 선착순 쿠폰 정확도 개선](https://oliveyoung.tech/2025-12-15/fcfs-coupon/) |
| `REQ.A.01.PP-010` | MQ 고부하로 후속 처리 중단 | 알림, 재처리, 보상 이벤트 큐는 queue lag, broker memory, consumer 처리량, DLQ를 운영 지표로 관리한다. | `REQ.A.01.FR-013`, `REQ.A.01.FR-015`, `REQ.A.01.FR-017` | `REQ.A.01.NFR-006`, `REQ.A.01.NFR-009`, `REQ.A.01.NFR-018` | [Oliveyoung RabbitMQ 장애](https://oliveyoung.tech/2025-10-28/coupon-mq-issue/) |
| `REQ.A.01.PP-011` | 이벤트 소비 병렬성과 DB 커넥션 경합 | consumer 병렬성, batch size, transaction propagation, DB pool 크기를 함께 산정해 피크 알림/후속 처리 timeout을 방지한다. | `REQ.A.01.FR-013`, `REQ.A.01.FR-017` | `REQ.A.01.NFR-006`, `REQ.A.01.NFR-009`, `REQ.A.01.NFR-013` | [Oliveyoung SQS 알림톡 데드락](https://oliveyoung.tech/2025-12-30/alimtalk_improve_event_driven_architecture/) |
| `REQ.A.01.PP-012` | 외부 PG 지연의 결제 서비스 전파 | PG별 timeout, bulkhead, queue 분리, 빠른 실패 기준을 두어 특정 결제수단 장애가 전체 빠른 결제로 번지지 않게 한다. | `REQ.A.01.FR-011`, `REQ.A.01.FR-012`, `REQ.A.01.FR-021` | `REQ.A.01.NFR-007`, `REQ.A.01.NFR-018`, `REQ.A.01.NFR-019` | [Mercari Payment Chaos Engineering](https://engineering.mercari.com/en/blog/entry/20211212-chaos-engineering-in-payment-service/) |
| `REQ.A.01.PP-013` | 통합 체크아웃의 변경 영향 범위 확대 | 빠른 결제와 일반 결제의 공통 요소는 일관되게 관리하되, 서비스별 확장 지점과 회귀 테스트 범위를 명확히 한다. | `REQ.A.01.FR-010`, `REQ.A.01.FR-011`, `REQ.A.01.FR-021` | `REQ.A.01.NFR-003`, `REQ.A.01.NFR-014`, `REQ.A.01.NFR-019` | [Mercari Flexible Checkout](https://engineering.mercari.com/en/blog/entry/20250617-building-a-flexible-checkout-solution-frontend-architecture-for-multi-service-integration/) |
| `REQ.A.01.PP-014` | 주문 상태 모델의 결합도 | 드롭 주문 상태를 재고 배정, 주문 생성, 결제 승인, 만료, 취소, 보상 단계로 분리해 상태 전이를 명확히 한다. | `REQ.A.01.FR-009`, `REQ.A.01.FR-011`, `REQ.A.01.FR-012`, `REQ.A.01.FR-018` | `REQ.A.01.NFR-004`, `REQ.A.01.NFR-012`, `REQ.A.01.NFR-013` | [Mercari Order Management](https://engineering.mercari.com/en/blog/entry/20251010-order-management-in-mercari-global-marketplace/) |
| `REQ.A.01.PP-015` | 카트·결제 시스템 교체의 서비스 경계 고민 | 카트, 주문, 결제, 재고를 무조건 쪼개기보다 MVP에서는 강한 정합성이 필요한 경계를 먼저 정의하고 MSA 분리 기준을 단계화한다. | `REQ.A.01.FR-008`, `REQ.A.01.FR-010`, `REQ.A.01.FR-011`, `REQ.A.01.FR-021` | `REQ.A.01.NFR-002`, `REQ.A.01.NFR-003`, `REQ.A.01.NFR-017`, `REQ.A.01.NFR-018` | [ZOZO 카트·결제 시스템 교체](https://techblog.zozo.com/entry/zozotown-cart-and-payment-system-modular-monolith-system-replacement) |
| `REQ.A.01.PP-016` | 로그인 강제가 탐색 전환을 막는 문제 | 내 정보나 결제 정보가 필요 없는 화면은 비로그인 상태에서도 접근할 수 있게 하고, 개인 정보가 필요한 화면에서만 로그인으로 보낸다. | `REQ.A.01.FR-001`, `REQ.A.01.FR-002`, `REQ.A.01.FR-022`, `REQ.A.01.FR-023` | `REQ.A.01.NFR-015`, `REQ.A.01.NFR-021` | TBD |

## 목표

- 사용자 목표: 사용자는 오픈 시간을 알고 기다리며, 같은 조건에서 참여하고, 구매 성공/실패 이유를 납득할 수 있다.
- 비즈니스 목표: 브랜드는 한정 상품 드롭을 이벤트처럼 운영하고, DropMong은 공정하고 신뢰 가능한 한정 판매 경험을 제공한다.
- 운영 목표: 운영자는 드롭 시작 전후의 트래픽, 재고, 주문, 결제, 알림, 실패율을 한 화면에서 확인하고 즉시 차단/완화/복구할 수 있다.
- 기술 목표: 재고 배정, 주문 생성, 결제 요청, 쿠폰/혜택, 알림 발송은 중복 요청과 비동기 실패에 안전해야 한다.

## 사용자 유형

| Actor ID | 사용자 | 목표 | 주요 행동 |
| --- | --- | --- | --- |
| `ACTOR-BUYER` | 구매자 | 원하는 한정 상품 드롭에 공정하게 참여한다. | 드롭 탐색, 알림 신청, 오픈 대기, 구매 시도, 결제, 결과 확인 |
| `ACTOR-BRAND-OPERATOR` | 브랜드 운영자 | 상품, 수량, 오픈 시간, 판매 조건을 등록하고 결과를 확인한다. | 드롭 생성, 상품/재고 등록, 판매 조건 설정, 현황 조회 |
| `ACTOR-DROPMONG-OPERATOR` | DropMong 운영자 | 드롭 오픈 순간의 안정성과 공정성을 관리한다. | 트래픽 감시, 차단/완화, 재처리, 공지, 인시던트 기록 |
| `ACTOR-CS` | CS 담당자 | 구매 실패, 중복 결제, 알림 누락 문의에 설명 가능한 답을 제공한다. | 사용자 이벤트 조회, 주문/결제/재고 상태 확인, 보상 정책 연결 |
| `ACTOR-ONCALL` | 온콜 담당자 | 피크 장애를 빠르게 선언하고 복구한다. | 알림 수신, 대시보드 확인, feature flag 조정, 장애 공유 |

## 기능 요구사항

| Req ID | 요구사항 | 사용자 | 우선순위 | 연결 Page/UC |
| --- | --- | --- | --- | --- |
| `REQ.A.01.FR-001` | 사용자는 홈 화면에서 현재 진행 중인 드롭 상품을 썸네일 카드로 보고, 예정된 드롭 목록을 시간순, 브랜드별, 관심 카테고리별로 탐색한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-002` | 사용자는 드롭 상세에서 오픈 시간, 판매 수량, 구매 제한, 배송/환불 조건, 참여 조건을 확인한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-003` | 사용자는 오픈 전 드롭 알림을 신청하고, 알림 수신 채널과 수신 상태를 확인한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-004` | 시스템은 드롭 오픈 전 카운트다운, 대기 상태, 오픈 임박 상태를 사용자에게 제공한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-005` | 시스템은 드롭 참여 전에 로그인, 휴대폰/이메일 인증, 구매 제한 조건, 차단 계정 여부를 확인한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-006` | 시스템은 오픈 시각 이후에만 구매 시도를 허용하고, 오픈 전 요청은 구매 처리로 넘기지 않는다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-007` | 시스템은 동일 사용자, 동일 상품, 동일 드롭에 대한 중복 구매 시도를 제한한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-008` | 시스템은 구매 시도 시점에 재고를 원자적으로 배정하거나 실패 사유를 반환한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-009` | 시스템은 재고 배정 후 제한 시간 안에 주문/결제를 완료하도록 하고, 만료 시 재고를 회수한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-010` | 사용자는 결제 전 상품, 가격, 배송비, 쿠폰/혜택, 최종 금액을 서버 계산 기준으로 확인한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-011` | 시스템은 주문 생성, 결제 요청, 결제 결과 반영을 idempotency key 기준으로 처리한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-012` | 사용자는 구매 성공, 품절, 대기 만료, 결제 실패, 조건 미충족 등 결과를 명확한 사유와 함께 확인한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-013` | 시스템은 드롭 오픈, 구매 성공, 결제 실패, 재고 회수, 배송 상태 변경을 이벤트로 기록하고 필요한 알림을 발송한다. | 구매자, CS | Must | TBD |
| `REQ.A.01.FR-014` | 브랜드 운영자는 드롭 상품, 오픈 시간, 판매 수량, 1인 구매 제한, 판매 노출 상태를 등록/수정한다. | 브랜드 운영자 | Must | TBD |
| `REQ.A.01.FR-015` | DropMong 운영자는 드롭별 트래픽, 참여자 수, 재고 배정 수, 주문 성공 수, 결제 실패 수, 알림 실패 수를 조회한다. | DropMong 운영자 | Must | TBD |
| `REQ.A.01.FR-016` | DropMong 운영자는 특정 드롭, 브랜드, API, 사용자 그룹에 대해 임시 차단 또는 읽기 전용 공지를 적용한다. | DropMong 운영자 | Must | TBD |
| `REQ.A.01.FR-017` | 시스템은 비동기 처리 실패를 DLQ 또는 재처리 큐에 남기고, 운영자가 재처리 또는 보상 처리를 수행할 수 있게 한다. | DropMong 운영자 | Must | TBD |
| `REQ.A.01.FR-018` | CS 담당자는 사용자 단위로 알림 신청, 구매 시도, 재고 배정, 주문, 결제, 실패 사유의 타임라인을 조회한다. | CS | Should | TBD |
| `REQ.A.01.FR-019` | 시스템은 찜, 알림 신청, 상세 조회, 검색, 구매 시도 로그를 드롭 성과 분석에 사용할 수 있게 저장한다. | 브랜드 운영자, DropMong 운영자 | Should | TBD |
| `REQ.A.01.FR-020` | 시스템은 추천/검색/노출 실험을 드롭 단위로 설정하고, 실험 결과를 동일한 metric 정의로 집계한다. | DropMong 운영자 | Could | TBD |
| `REQ.A.01.FR-021` | 사용자는 기본 등록된 결제수단으로 빠른 결제를 수행한다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-022` | 사용자는 로그인하지 않아도 홈, 드롭 목록, 드롭 상세, 검색, 공지처럼 내 정보나 결제 정보가 필요 없는 화면을 탐색할 수 있다. | 구매자 | Must | TBD |
| `REQ.A.01.FR-023` | 시스템은 로그인하지 않은 사용자가 내 정보, 주문 내역, 결제수단, 빠른 결제, 구매 시도처럼 개인 정보나 결제 정보가 필요한 화면에 진입하면 로그인 페이지로 이동시킨다. | 구매자 | Must | TBD |

## 비기능 요구사항

| Req ID | 요구사항 | 기준 | 연결 문서 |
| --- | --- | --- | --- |
| `REQ.A.01.NFR-001` | 드롭 오픈 시각 판정은 서버 시간을 기준으로 한다. | 클라이언트 시간이 달라도 오픈 전 구매 처리가 발생하지 않는다. | TBD |
| `REQ.A.01.NFR-002` | 재고 배정은 동시 요청에 대해 원자적으로 처리한다. | 판매 가능 수량보다 많은 성공 배정이 발생하지 않는다. | TBD |
| `REQ.A.01.NFR-003` | 주문/결제 생성은 중복 요청에 안전해야 한다. | 같은 idempotency key는 같은 결과를 반환하고 중복 결제를 만들지 않는다. | TBD |
| `REQ.A.01.NFR-004` | 사용자에게 보인 성공과 최종 처리 성공을 별도 상태로 추적한다. | 성공 응답 후 비동기 실패가 발생하면 보상/재처리 대상에 남는다. | TBD |
| `REQ.A.01.NFR-005` | 피크 트래픽은 핵심 쓰기 경로와 조회 경로를 분리해 처리한다. | 드롭 상세/카운트다운 조회 폭증이 재고 배정/주문 생성 DB를 압박하지 않는다. | TBD |
| `REQ.A.01.NFR-006` | 드롭 오픈 순간의 주요 API는 관측 가능해야 한다. | p50/p95/p99 latency, error rate, saturation, queue lag, 재고 배정 성공률을 드롭별로 확인한다. | TBD |
| `REQ.A.01.NFR-007` | 외부 시스템 장애는 사용자 경험으로 무제한 전파되지 않아야 한다. | PG, 알림, 배송, 추천 장애 시 fallback, retry, circuit breaker, degraded mode 기준을 둔다. | TBD |
| `REQ.A.01.NFR-008` | 운영자는 피크 중에도 특정 드롭 또는 기능을 빠르게 차단할 수 있어야 한다. | 배포 없이 feature flag 또는 운영 설정으로 차단/공지/읽기 전용 전환이 가능하다. | TBD |
| `REQ.A.01.NFR-009` | 알림 발송은 중복과 누락을 추적할 수 있어야 한다. | 알림 요청, 공급자 응답, 실패, 재시도, 최종 상태가 이벤트로 남는다. | TBD |
| `REQ.A.01.NFR-010` | 구매 제한과 봇 방어 정책은 공정성을 해치지 않게 감사 가능해야 한다. | rate limit, 계정 제한, 디바이스/네트워크 제한의 적용 사유와 해제 이력이 남는다. | TBD |
| `REQ.A.01.NFR-011` | 가격, 쿠폰, 배송비는 서버 계산 결과를 기준으로 확정한다. | 클라이언트 금액은 검증용으로만 사용하고, 결제 전 서버 스냅샷을 저장한다. | TBD |
| `REQ.A.01.NFR-012` | 드롭 운영 데이터는 사후 분석과 CS 대응에 충분해야 한다. | 사용자별 타임라인과 드롭별 집계가 같은 원천 이벤트에서 재구성된다. | TBD |
| `REQ.A.01.NFR-013` | 데이터 변경과 메시지 발행의 순서가 어긋나지 않아야 한다. | 주문/재고 상태 변경 후 이벤트 발행은 outbox 또는 after-commit 기준을 따른다. | TBD |
| `REQ.A.01.NFR-014` | 대규모 마이그레이션과 정책 변경은 점진 적용해야 한다. | shadow mode, side-by-side, feature flag, rollback 기준 없이 핵심 구매 경로를 교체하지 않는다. | TBD |
| `REQ.A.01.NFR-015` | 개인정보와 인증 정보는 최소 보관과 암호화를 기본값으로 한다. | 인증/연락처/결제 관련 민감 정보는 접근 권한, 암호화, 감사 로그 기준을 갖는다. | TBD |
| `REQ.A.01.NFR-016` | 드롭 결과 화면은 실패 상황에서도 사용자가 다음 행동을 알 수 있게 한다. | 품절, 대기 만료, 결제 실패, 차단, 점검 상태에 대해 다른 메시지와 추적 ID를 제공한다. | TBD |
| `REQ.A.01.NFR-017` | 시스템은 MSA 서비스로 구성하고 무중단 배포를 지원해 배포 중에도 서비스가 중단되지 않도록 한다. | rolling update, readiness/liveness check, backward-compatible API, database migration 분리, 빠른 rollback 기준을 갖는다. | TBD |
| `REQ.A.01.NFR-018` | 특정 서비스에 장애가 발생하더라도 전체 서비스 품질에 영향을 주지 않도록 장애를 격리한다. | service timeout, bulkhead, circuit breaker, fallback, queue 기반 비동기 처리로 장애 서비스의 영향 범위를 제한한다. | TBD |
| `REQ.A.01.NFR-019` | 기본 결제수단 기반 빠른 결제는 피크 상황에서도 짧은 응답 시간을 유지해야 한다. | 기본 결제수단 조회, 주문 금액 검증, 결제 요청 생성 경로는 캐시/사전 검증/timeout 기준을 두고 p95 latency 목표를 별도로 관리한다. | TBD |
| `REQ.A.01.NFR-020` | 저장된 결제수단은 안전하게 보관하고 결제 직전 유효성을 검증해야 한다. | 카드 원문 정보는 저장하지 않고 PG token 또는 결제수단 식별자만 사용하며, 만료/해지/추가 인증 필요 상태는 빠른 결제 전에 감지한다. | TBD |
| `REQ.A.01.NFR-021` | 인증 게이트는 화면과 API의 정보 민감도에 맞춰 일관되게 적용해야 한다. | 공개 탐색 화면은 인증 없이 응답하고, 개인/결제 정보 화면과 API는 인증 실패 시 로그인 이동 또는 `401`을 반환하며 원래 가려던 위치를 보존한다. | TBD |

## 제약 조건

- 정책/법적 제약: 개인정보 보호, 전자상거래 고지, 청약철회/환불, 결제 기록 보존, 광고성 알림 수신 동의를 준수해야 한다.
- 기술 제약: 드롭 오픈 순간에는 읽기 트래픽과 쓰기 트래픽이 동시에 증가하므로, 단일 RDB 트랜잭션만으로 전체 경험을 처리하면 병목과 lock 위험이 커진다.
- 일정 제약: 초기 버전은 검색/추천 고도화보다 드롭 생성, 알림 신청, 공정한 참여, 재고 배정, 주문/결제, 운영 관측을 우선한다.
- 데이터 제약: 브랜드 공급 재고, 결제 승인, 알림 provider 응답은 외부 시스템 상태에 의존한다.
- 운영 제약: 피크 중 수동 DB 수정이나 배포만으로 대응하는 구조는 허용하지 않는다. 운영 설정, 재처리, 차단, 공지 경로를 별도로 둔다.

## 수용 기준

- 오픈 전 구매 요청은 모두 실패하며, 실패 사유가 `NOT_OPENED`처럼 추적 가능한 코드로 남는다.
- 홈 화면에는 현재 진행 중인 드롭 상품의 썸네일, 브랜드명, 상품명, 남은 시간 또는 진행 상태가 표시된다.
- 판매 수량이 `N`인 드롭에서 동시 구매 시도 후 성공 재고 배정 수는 `N`을 초과하지 않는다.
- 동일 사용자의 동일 드롭 중복 요청은 중복 주문/중복 결제를 만들지 않는다.
- 결제 성공 후 주문 반영 실패, 알림 실패, 재고 회수 실패는 운영 재처리 목록에 남는다.
- 기본 결제수단이 정상인 사용자는 드롭 구매 화면에서 추가 결제수단 선택 없이 빠른 결제를 시도할 수 있다.
- 기본 결제수단이 만료되었거나 추가 인증이 필요한 경우 빠른 결제는 실패 사유를 표시하고 일반 결제 경로로 전환한다.
- 비로그인 사용자는 홈, 드롭 목록, 드롭 상세, 검색, 공지 화면을 둘러볼 수 있다.
- 비로그인 사용자가 내 정보, 주문 내역, 결제수단, 빠른 결제, 구매 시도 화면에 진입하면 로그인 페이지로 이동하고 로그인 후 원래 위치로 돌아갈 수 있다.
- 운영자는 드롭별로 요청량, 성공/실패 수, 품절 시각, 결제 실패율, 알림 실패율을 확인할 수 있다.
- 특정 드롭을 배포 없이 임시 차단하고 사용자에게 점검/혼잡 안내를 보여줄 수 있다.
- 배포 중에도 드롭 조회, 구매 시도, 주문/결제 상태 확인의 사용자 요청이 중단 없이 처리된다.
- 특정 서비스 장애가 발생해도 장애 서비스에 직접 의존하지 않는 홈, 드롭 상세, 주문 상태 조회 같은 핵심 기능은 계속 동작한다.
- CS 담당자는 사용자 문의에 대해 "언제 알림을 신청했고, 언제 구매를 시도했고, 왜 실패했는지"를 이벤트 타임라인으로 확인할 수 있다.

## 연관 태그

🏷️ 플로우 참조: FLOW.A.01 | 페이지 참조: TBD | UI 참조: TBD | UC 참조: TBD | 영속성 참조: TBD | 서비스 참조: TBD | 시나리오 참조: TBD | API 참조: TBD

## 열린 질문

- 드롭 참여 방식은 즉시 선착순, 대기열, 추첨형 응모 중 무엇을 1차 정책으로 둘 것인가?
- 재고 배정의 유효 시간은 상품/브랜드별로 다르게 둘 것인가, 전역 정책으로 둘 것인가?
- 봇 방어를 계정, 디바이스, IP, 결제수단, 행동 패턴 중 어디까지 적용할 것인가?
- 구매 실패 사용자를 위한 후속 경험은 단순 품절 안내, 재입고 알림, 다음 드롭 추천 중 어디까지 제공할 것인가?
- 브랜드 운영자가 직접 드롭을 열 수 있는가, 아니면 DropMong 운영자 승인 후 오픈되는가?
- 결제 실패 후 재고를 즉시 회수할지, 짧은 유예 시간을 줄지 결정해야 한다.

## 확인 필요

- PG provider별 idempotency, 결제 취소, 부분 승인 실패, webhook 재시도 정책
- 알림 provider별 rate limit, 실패 코드, 재시도 가능 여부
- 초기 예상 드롭 규모: 동시 접속자, 판매 수량, 초당 구매 시도, 알림 신청자 수
- 브랜드별 판매 조건: 1인 1개, 옵션별 수량, 세트 상품, 쿠폰 동시 적용 가능 여부
- 운영 대시보드의 최소 지표와 온콜 알림 기준

## 새로 드러난 가장자리

- 이 자료가 답한 질문: 주요 버티컬 이커머스의 공개 기술 글에서 반복되는 피크 트래픽, 정합성, 관측성, 운영 차단, 점진 이전 문제를 DropMong 요구사항으로 변환했다.
- 아직 남은 질문: DropMong의 1차 드롭 참여 정책이 선착순인지, 대기열인지, 추첨인지 아직 정해지지 않았다.
- 검증되지 않은 전제: 드롭 상품은 대부분 단일 브랜드/단일 이벤트 단위로 운영된다고 가정했다.
- 약한 연결: 검색/추천/실험 플랫폼 요구는 초기 MVP보다 후순위일 가능성이 있어, 실제 서비스 범위에 맞춰 우선순위를 조정해야 한다.
- 다음에 확인할 것: 예상 동시 접속 규모, PG/알림 provider, 브랜드 운영 권한, 재고 배정 유효 시간, 실패 보상 정책

## 레퍼런스

- Kurly: [쿠폰과 할인으로 앞다리살 하나 더 판매한 이야기](https://helloworld.kurly.com/blog/%EC%BF%A0%ED%8F%B0%EA%B3%BC-%ED%95%A0%EC%9D%B8%EC%9C%BC%EB%A1%9C-%EC%95%9E%EB%8B%A4%EB%A6%AC%EC%82%B4-%ED%95%98%EB%82%98-%EB%8D%94-%ED%8C%90%EB%A7%A4%ED%95%9C-%EC%9D%B4%EC%95%BC%EA%B8%B0), [카트 개발 연대기](https://helloworld.kurly.com/blog/my-cart-development-history), [OMS의 최적화된 마이크로서비스 아키텍처 디자인](https://helloworld.kurly.com/blog/oms-msa-architecture-1), [nginx 설정 없이 우아하게 서비스 점검하기](https://helloworld.kurly.com/blog/access-block-1)
- Oliveyoung: [올리브영 결제 이야기 Part - 3](https://oliveyoung.tech/2022-12-13/oliveyoung-transaction-orderstock/), [CircuitBreaker를 사용한 장애 전파 방지](https://oliveyoung.tech/2023-08-31/circuitbreaker-inventory-squad/), [Kafka Streams 기반 EDA 구축 사례](https://oliveyoung.tech/2025-12-15/kafka-streams-for-out-of-stock/), [선착순 쿠폰 시스템 정확도 개선기](https://oliveyoung.tech/2025-12-15/fcfs-coupon/), [RabbitMQ Classic Queue 메모리 장애와 Quorum Queue 전환기](https://oliveyoung.tech/2025-10-28/coupon-mq-issue/), [SQS 기반 알림톡 처리에서 발생한 DB 커넥션 데드락 분석기](https://oliveyoung.tech/2025-12-30/alimtalk_improve_event_driven_architecture/), [올리브영은 인시던트를 어떻게 관리하고 있는가?](https://oliveyoung.tech/2024-01-23/incident/)
- Bucketplace: [신뢰할 수 있는 메트릭과 실험 플랫폼](https://www.bucketplace.com/post/2026-02-06-%EC%8B%A0%EB%A2%B0%ED%95%A0-%EC%88%98-%EC%9E%88%EB%8A%94-%EB%A9%94%ED%8A%B8%EB%A6%AD%EA%B3%BC-%EC%8B%A4%ED%97%98-%ED%94%8C%EB%9E%AB%ED%8F%BC/), [개인화 추천 시스템 4 Feature Store](https://www.bucketplace.com/post/2025-12-17-%EA%B0%9C%EC%9D%B8%ED%99%94-%EC%B6%94%EC%B2%9C-%EC%8B%9C%EC%8A%A4%ED%85%9C-4-feature-store/), [검색/디스플레이 광고 내재화 프로젝트](https://www.bucketplace.com/post/2025-12-12-%EA%B2%80%EC%83%89/%EB%94%94%EC%8A%A4%ED%94%8C%EB%A0%88%EC%9D%B4-%EA%B4%91%EA%B3%A0-%EB%82%B4%EC%9E%AC%ED%99%94-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8/)
- Mercari: [Chaos engineering with chaos mesh in payment-service](https://engineering.mercari.com/en/blog/entry/20211212-chaos-engineering-in-payment-service/), [Building a Flexible Checkout Solution](https://engineering.mercari.com/en/blog/entry/20250617-building-a-flexible-checkout-solution-frontend-architecture-for-multi-service-integration/), [Order Management in Mercari Global Marketplace](https://engineering.mercari.com/en/blog/entry/20251010-order-management-in-mercari-global-marketplace/), [From DNS Failures to Resilience](https://engineering.mercari.com/en/blog/entry/20250515-from-dns-failures-to-resilience-how-nodelocal-dnscache-saved-the-day/), [The Cost of Speed](https://engineering.mercari.com/en/blog/entry/20251215-the-cost-of-speed-a-battle-against-cost-debt-and-diverging-systems/), [Rethinking Payment APIs at Merpay](https://engineering.mercari.com/en/blog/entry/20260626-rethinking-payment-apis-at-merpay-the-exchange-abstraction/)
- ZOZO: [ZOZOTOWNカート決済システムリプレイス](https://techblog.zozo.com/entry/zozotown-cart-and-payment-system-modular-monolith-system-replacement), [検索ログ分析](https://techblog.zozo.com/entry/unclicked-products-reappearing-analysis), [お気に入りDB無停止移行](https://techblog.zozo.com/entry/favorite-api-data-migration), [Push通知エラートークン管理](https://techblog.zozo.com/entry/fcm-token-management-refinement)
