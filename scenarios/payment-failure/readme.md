# 결제 실패 시나리오

작성일: 2026-07-07

이 폴더는 DropMong에서 주문 예약 후 결제가 실패하거나 지연될 때 사용자가 주문 상태를 이해할 수 있도록 서비스, 이벤트, 데이터, 테스트 기준을 정의한다. 기준 문서는 `../../medikong/12-user-flows.md`의 "결제 실패 시나리오 오너"와 `../../blueprint/10-sitemap/buyer-mobile-web/PAGE_A_11_payment.md`의 결제 실패/진행 중 화면 상태다.

## 목표

결제 실패, 결제 지연, 예약 만료 상황에서 중복 결제를 만들지 않고 예약 재고와 주문 상태를 일관되게 정리한다.

```text
주문 예약 성공
-> 결제 실패 또는 지연
-> payment event
-> order-service 보상 전이
-> 예약 재고 release 또는 만료
-> 사용자 상태 확인
```

## 문서

| 문서 | 내용 |
| --- | --- |
| `00-detailed-design.md` | 결제 실패, 결제 지연, 예약 만료의 사용자 흐름, API, 이벤트, 데이터, 테스트 상세 설계 |
| `01-user-journey.md` | 실패·지연·만료 시 사용자에게 보이는 상태와 다음 행동 |
| `02-api-flow.md` | 현재 mock 실패 API와 목표 결제 API, Kafka 이벤트 경계 |
| `03-state-event-flow.md` | 현재 `PAYMENT_FAILED`와 목표 `CANCELED`·`EXPIRED` 상태 전이 |
| `04-service-implementation-plan.md` | 서비스별 현재 구현 위치와 후속 구현 단위 |
| `05-test-scenarios.md` | 현재 gate와 후속 수용 테스트 시나리오 |
| `test-execution-record.md` | 현재 구현 상태와 실제로 실행한 단위 테스트, E2E 테스트, 추가해야 할 통합 테스트 기록 |

## 현재 완료 범위

- 완료: mock 결제 실패, 실패 결제 1행 저장, `payment.failed` 중복 처리, 주문 `PAYMENT_FAILED`, 실패 주문의 예약 집계 제외
- 미완료: 결제 지연, `CANCELED`, `EXPIRED`, 늦은 승인, 실패 알림, transactional outbox

목표 설계와 현재 구현이 다를 때는 `test-execution-record.md`와 `../_shared/03-purchase-development-handoff.md`의 현재 구현을 우선 확인한다. 기계 계약의 취소 상태 표기는 `CANCELED`다.

## 관련 시나리오

| 시나리오 | 관계 |
| --- | --- |
| 정상 구매 | 결제 승인 성공은 정상 구매의 `PENDING_PAYMENT -> CONFIRMED` 흐름을 사용한다. |
| 품절/동시성 | 결제 실패와 만료는 예약 재고를 release해서 이후 주문 가능 수량에 영향을 준다. |
