# DropMong 시나리오 상세 설계

작성일: 2026-07-07

이 폴더는 `../medikong/12-user-flows.md`와 `../blueprint/`를 구현 가능한 시나리오 단위로 풀어낸다. `12-user-flows.md`가 사용자 흐름의 큰 뼈대라면, 이 폴더는 서비스, API, 이벤트, 데이터, 테스트, 인프라 확인점을 시나리오별 개발 작업으로 바꾼 문서다.

## 기준 문서

| 기준 | 문서 | 역할 |
| --- | --- | --- |
| 사용자 흐름 | `../medikong/12-user-flows.md` | 정상 구매, 품절/동시성, 결제 실패 흐름의 기준 |
| 제품 요구사항 | `../blueprint/00-requirements/REQ_A_01_limited_drop_commerce.md` | 한정 드롭 커머스 요구사항, 비기능 요구사항, 수용 기준 |
| 구매 유스케이스 | `../blueprint/30-uc/UC_A_01_buyer_purchase_delivery.md` | 구매자의 발견, 주문, 결제, 결과 확인 흐름 |
| 상품 상세 | `../blueprint/10-sitemap/PAGE_A_02_product_detail.md` | 오픈 전/오픈 중/품절 화면 상태 |
| 주문/결제 | `../blueprint/10-sitemap/PAGE_A_11_payment.md` | 주문 확정 전 최종 검증과 결제 실패 화면 상태 |
| 공통 계약 | `_shared/00-shared-infra-test-contract.md` | API, Kafka topic, DB 소유권, JWT, 테스트 이름 |
| 관측성 테스트 | `_shared/01-observability-driven-test-contract.md` | trace, metric, log, Kafka lag를 이용한 실전형 테스트 기준 |

## 이번 개발 대상

| 시나리오 | 폴더 | 목표 |
| --- | --- | --- |
| 정상 구매 | `normal-purchase/` | 로그인된 고객이 드롭을 조회하고 결제 승인 후 주문 확정과 알림까지 확인한다. |
| 품절/동시성 | `sold-out-concurrency/` | 동시에 몰린 주문에서 재고 수량을 초과한 성공이 발생하지 않고, 실패 사용자는 납득 가능한 결과를 받는다. |
| 결제 실패 | `payment-failure/` | 결제 실패와 지연 상황에서 주문, 예약 재고, 사용자 안내가 일관되게 정리된다. |

각 시나리오 폴더의 `test-execution-record.md`는 현재 구현과 실제 테스트 실행 기록을 추적한다. 설계 문서는 무엇을 만들어야 하는지, 실행 기록 문서는 무엇을 확인했는지를 분리해서 관리한다.

## 구현 순서 권장

```text
정상 구매
-> 주문 재고 예약 원자화
-> 품절/동시성
-> 결제 실패
-> 결제 지연/예약 만료
```

정상 구매는 happy path의 API와 이벤트를 고정한다. 품절/동시성은 `order-service`의 재고 예약 원장을 실제 경쟁 상황에 견딜 수 있게 만든다. 결제 실패는 `payment-service` 이벤트와 `order-service` 보상 전이를 붙인다.

## 공통 결정

| 항목 | 결정 |
| --- | --- |
| 인증 | 구매, 결제, 주문 조회, 알림 조회는 Istio Ingress Gateway JWT 검증 이후 내부 `X-User-*` context를 사용한다. |
| API | 외부 계약은 REST + JSON + OpenAPI로 유지한다. |
| 이벤트 | 상태 변경 전파는 Kafka topic을 사용한다. |
| DB | 서비스별 DB 소유권을 지키고 다른 서비스 DB 직접 접근은 금지한다. |
| 재고 진실 | `order-service`가 소유한다. |
| 결제 진실 | `payment-service`는 결제 사실을 이벤트로 발행하고, 주문 상태 전이는 `order-service`가 판단한다. |
| 알림 | checkout 성공/실패 판단의 동기 경로에 넣지 않고 비동기로 처리한다. |

## 구현 전 계약 정렬 필요

현재 설계 문서 기준과 일부 구현 초안 사이에 이름 차이가 있을 수 있다. 신규 개발은 이 폴더와 `_shared/00-shared-infra-test-contract.md`를 우선한다.

| 항목 | 기준 |
| --- | --- |
| 결제 API | `POST /payments` |
| drop 상태 | `SCHEDULED`, `OPEN`, `SOLD_OUT`, `CLOSED` |
| order 상태 | `PENDING_PAYMENT`, `CONFIRMED`, `CANCELLED`, `EXPIRED` |
| payment 상태 | `REQUESTED`, `APPROVED`, `FAILED`, `DELAYED` |
