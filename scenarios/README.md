# DropMong 시나리오 상세 설계

작성일: 2026-07-07

이 폴더는 `../medikong/12-user-flows.md`와 `../blueprint/`를 구현 가능한 시나리오 단위로 풀어낸다. `12-user-flows.md`가 사용자 흐름의 큰 뼈대라면, 이 폴더는 서비스, API, 이벤트, 데이터, 테스트, 인프라 확인점을 시나리오별 개발 작업으로 바꾼 문서다.

## 기준 문서

| 기준 | 문서 | 역할 |
| --- | --- | --- |
| 사용자 흐름 | `../medikong/12-user-flows.md` | 정상 구매, 품절/동시성, 결제 실패 흐름의 기준 |
| 제품 요구사항 | `../blueprint/00-requirements/REQ_A_01_limited_drop_commerce.md` | 한정 드롭 커머스 요구사항, 비기능 요구사항, 수용 기준 |
| 구매 유스케이스 | `../blueprint/30-uc/UC_A_01_buyer_purchase_delivery.md` | 구매자의 발견, 주문, 결제, 결과 확인 흐름 |
| 상품 상세 | `../blueprint/10-sitemap/buyer-mobile-web/PAGE_A_02_product_detail.md` | 오픈 전/오픈 중/품절 화면 상태 |
| 주문/결제 | `../blueprint/10-sitemap/buyer-mobile-web/PAGE_A_11_payment.md` | 주문 확정 전 최종 검증과 결제 실패 화면 상태 |
| 공통 계약 | `_shared/00-shared-infra-test-contract.md` | API, Kafka topic, DB 소유권, JWT, 테스트 이름 |
| 관측성 테스트 | `_shared/01-observability-driven-test-contract.md` | trace, metric, log, Kafka lag를 이용한 실전형 테스트 기준 |
| Docker E2E 검증 | `_shared/02-docker-purchase-e2e-runbook.md` | 정상 구매, 결제 실패, 품절/동시성 반복 실행과 metric 확인 기준 |
| 팀 개발 인수인계 | `_shared/03-purchase-development-handoff.md` | 서비스 책임, 코드 구조, API·Kafka 흐름, 개발 순서와 다음 작업 |
| 통합 진행 현황 | `progress.md` | 7개 Task의 현재 상태, 완료 증거, 다음 행동을 누적 추적 |

## 문서 구조

`blueprint`는 제품과 서비스의 목표 설계를 관리하고, 이 폴더는 목표를 개발 가능한 시나리오로 분해해 현재 구현과 실행 증거에 연결한다. 시나리오 문서의 API·이벤트·상태 표는 설계 초안 또는 설명이며 기계 계약이 아니다. 권위 있는 계약은 OpenAPI와 이벤트 계약이고, 실제 완료 상태는 실행 기록과 인수인계 문서를 기준으로 한다.

각 시나리오는 같은 순서로 읽는다.

| 순서 | 문서 | 역할 |
| ---: | --- | --- |
| 1 | `README.md` 또는 `readme.md` | 범위, 현재 상태, 읽는 순서 |
| 2 | `00-detailed-design.md` | blueprint 요구사항을 구현 후보와 수용 기준으로 연결한 목표 설계 초안 |
| 3 | `01-user-journey.md` | 사용자에게 보이는 행동과 결과 |
| 4 | `02-api-flow.md` | REST와 Kafka 경계, 계약 원장 링크 |
| 5 | `03-state-event-flow.md` | 상태, 데이터, transaction, 멱등성 |
| 6 | `04-service-implementation-plan.md` | 서비스별 책임과 실제 코드 시작점 |
| 7 | `05-test-scenarios.md` | Given/When/Then 수용 테스트와 gate 연결 |
| 8 | `test-execution-record.md` | 실제 명령, 결과, 실패 기록, 잔여 위험 |

세 시나리오와 blueprint의 연결은 `_shared/04-blueprint-traceability.md`에서 한 번에 확인한다. 서비스 언어와 REST/gRPC/Kafka 경계처럼 여러 시나리오에 영향을 주는 제안은 `_shared/adr/`에서 상태와 함께 관리하며, `제안` 상태 ADR은 확정 계약으로 해석하지 않는다.

## 이번 개발 대상

| 시나리오 | 폴더 | 목표 |
| --- | --- | --- |
| 정상 구매 | `normal-purchase/` | 로그인된 고객이 드롭을 조회하고 결제 승인 후 주문 확정과 알림까지 확인한다. |
| 품절/동시성 | `sold-out-concurrency/` | 동시에 몰린 주문에서 재고 수량을 초과한 성공이 발생하지 않고, 실패 사용자는 납득 가능한 결과를 받는다. |
| 결제 실패 | `payment-failure/` | 결제 실패와 지연 상황에서 주문, 예약 재고, 사용자 안내가 일관되게 정리된다. |

각 시나리오 폴더의 `test-execution-record.md`는 현재 구현과 실제 테스트 실행 기록을 추적한다. 설계 문서는 blueprint를 바탕으로 한 목표와 제안을 설명하고, 실행 기록 문서는 실제로 확인한 동작을 관리한다. 둘이 다르면 실행 기록과 공통 인수인계 문서가 현재 상태의 기준이다.

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

아래 표는 현재 구매 시나리오가 사용하는 구조적 기준이다. 서비스 언어 전환과 새 gRPC 경계는 별도 ADR이 채택되기 전까지 제안으로만 취급한다.

| 항목 | 결정 |
| --- | --- |
| 인증 | 구매, 결제, 주문 조회, 알림 조회는 Istio Ingress Gateway JWT 검증 이후 내부 `X-User-*` context를 사용한다. |
| API | 외부 계약은 REST + JSON + OpenAPI로 유지한다. |
| 이벤트 | 상태 변경 전파는 Kafka topic을 사용한다. |
| DB | 서비스별 DB 소유권을 지키고 다른 서비스 DB 직접 접근은 금지한다. |
| 재고 진실 | `order-service`가 소유한다. |
| 결제 진실 | `payment-service`는 결제 사실을 이벤트로 발행하고, 주문 상태 전이는 `order-service`가 판단한다. |
| 알림 | checkout 성공/실패 판단의 동기 경로에 넣지 않고 비동기로 처리한다. |

## 설계 기준과 현재 구현 차이

설계 문서는 목표 상태를 정의하고 현재 코드는 내부 구매 시나리오를 먼저 검증한 상태다. 신규 개발은 `_shared/03-purchase-development-handoff.md`에서 현재 구현 차이를 확인한 뒤 계약을 변경한다.

| 항목 | 설계 기준 | 현재 구현 |
| --- | --- | --- |
| 결제 API | `POST /payments` | `/payments/mock-approvals`, `/payments/mock-failures` |
| drop 상태 | `SCHEDULED`, `OPEN`, `SOLD_OUT`, `CLOSED` | fixture는 `UPCOMING`, `OPEN`, `SOLD_OUT`, `CLOSED` |
| order 상태 | `PENDING_PAYMENT`, `CONFIRMED`, `CANCELED`, `EXPIRED` | 자동 검증은 `PENDING_PAYMENT`, `CONFIRMED`, `PAYMENT_FAILED` 중심 |
| payment 상태 | `REQUESTED`, `APPROVED`, `FAILED`, `DELAYED` | 자동 검증은 `APPROVED`, `FAILED` 중심 |
| 인증 | Istio JWT 검증 후 내부 context | 현재 구매 서비스와 E2E는 `X-User-Id`, `X-User-Role`을 직접 사용 |
