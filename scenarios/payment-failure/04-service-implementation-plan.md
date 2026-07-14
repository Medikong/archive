# 결제 실패 서비스 구현 계획

작성일: 2026-07-14

이 문서는 현재 코드 시작점과 후속 구현 단위를 연결한다. 경로는 `Medikong/services` 저장소 루트를 기준으로 한다.

## 1. 현재 구현

| 서비스 | 현재 책임 | 코드 시작점 |
| --- | --- | --- |
| payment-service | mock 실패 결제, REST 멱등성, `payment.failed` 발행 | `services/payment-service/app/main.py`, `postgres.py`, `messaging.py` |
| order-service | 실패 event inbox, `PAYMENT_FAILED`, 활성 예약 제외 | `services/order-service/app/postgres.py`, `messaging.py` |
| notification-service | 현재 실패 알림 미지원 | `services/notification-service/app/messaging.py` |

## 2. 후속 구현 단위

| 순서 | 작업 | 완료 조건 |
| ---: | --- | --- |
| 1 | 상태 계약 확정 | `PAYMENT_FAILED`, `CANCELED`, `EXPIRED`의 전이와 API 표현이 일치한다. |
| 2 | transactional outbox | 결제 저장과 `payment.failed`, 주문 전이와 후속 event가 원자적으로 기록된다. |
| 3 | 결제 지연과 만료 | `DELAYED`, TTL worker, `EXPIRED`가 실제 DB와 E2E에서 검증된다. |
| 4 | 늦은 승인 | 만료·취소 주문을 확정하지 않고 reconciliation 증거를 남긴다. |
| 5 | 실패 알림 | 실패·만료 알림이 event ID 기준으로 중복 생성되지 않는다. |
| 6 | retry와 DLQ | poison event와 일시 장애의 복구 경로가 자동 검증된다. |

## 3. 구현 원칙

- payment-service는 결제 사실을 소유하고 주문 상태를 직접 변경하지 않는다.
- order-service가 주문 전이와 예약 활성 여부를 최종 판단한다.
- 상태와 이벤트 이름은 OpenAPI와 이벤트 계약을 먼저 변경한다.
- Go 전환 여부와 무관하게 외부 REST와 Kafka 계약을 유지한다.
- 공통 패키지는 두 서비스 이상에서 실제 같은 동작이 필요할 때만 확장한다.

## 4. 회귀 기준

후속 작업은 `task payment-failure-idempotency`와 `task purchase-internal-regression`을 모두 통과해야 한다. Gateway JWT는 별도 경계로 유지한다.
