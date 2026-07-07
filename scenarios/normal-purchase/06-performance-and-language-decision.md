# 성능과 서비스 언어 결정 기준

작성일: 2026-07-03

이 문서는 DropMong 정상 구매 시나리오에서 FastAPI, Go, 다른 언어를 어떤 기준으로 선택할지 정의한다. 언어 선택은 서비스 계약을 바꾸는 이유가 아니며, API와 이벤트 계약은 언어와 독립적으로 유지한다.

## 1. 기본 결정

```text
기본 언어는 FastAPI로 한다.
성능 병목이 명확한 서비스는 Go를 우선 검토한다.
다른 언어는 명확한 도입 조건이 생길 때만 검토한다.
```

## 2. 서비스별 기준

| 서비스 | 기본 선택 | 전환 후보 | 이유 |
| --- | --- | --- | --- |
| `auth-service` | FastAPI | 없음 | 인증과 JWT 발급 중심이며 기존 구현 재사용 가치가 높다. |
| `catalog-service` | FastAPI | 없음 | 조회 성능은 언어보다 캐시, index, read model 영향이 크다. |
| `order-service` | Go 후보 | Go 우선 검토 | 동시 주문, DB lock, 재고 예약, idempotency가 critical path이다. |
| `payment-service` | FastAPI | Java/Kotlin 후보 | 실제 PG/정산/환불이 커질 때만 재검토한다. |
| `notification-service` | FastAPI | Go worker 후보 | Kafka lag가 병목이면 consumer만 Go로 분리할 수 있다. |
| `coupon-service` | Go 후보 | Go | 선착순 발급과 카운터 경쟁이 크면 Go가 적합하다. |
| `message-recovery-service` | FastAPI | 없음 | 운영자 조회와 감사 로그가 중심이다. |

## 3. Go 전환 검토 조건

아래 조건 중 하나라도 반복되면 Go 전환 또는 Go 유지 결정을 ADR로 기록한다.

| 조건 | 기준 |
| --- | --- |
| latency | `POST /orders` admitted p95가 목표치를 지속적으로 초과한다. |
| 동시성 | 재고 1개에 동시 주문이 몰릴 때 transaction 대기와 timeout이 증가한다. |
| CPU | Python worker를 늘려도 CPU 대비 처리량이 충분히 늘지 않는다. |
| memory | pod당 메모리 사용량이 HPA 효율을 떨어뜨린다. |
| 이벤트 처리 | Kafka consumer lag가 scale-out만으로 줄지 않는다. |
| 비용 | FastAPI replica 증가보다 Go 전환이 운영 비용을 낮춘다는 테스트 근거가 있다. |

## 4. 다른 언어 검토 조건

| 언어 | 검토 조건 | 지금 판단 |
| --- | --- | --- |
| TypeScript/Node.js | BFF, admin 화면, frontend와 타입 공유가 필요할 때 | 지금은 보류 |
| Java/Kotlin | 실제 PG 연동, 정산, 환불, 회계성 workflow가 커질 때 | payment/settlement 후보 |
| Rust | admission, rate limiter처럼 초저지연 고성능 컴포넌트가 필요할 때 | 지금은 과함 |
| Elixir | 실시간 대기열, WebSocket 상태 동기화가 핵심 기능이 될 때 | 지금은 보류 |

## 5. 통신 방식 결정

| 통신 | 기준 |
| --- | --- |
| REST | 외부 API와 단순 동기 조회에 사용한다. |
| Kafka | 주문, 결제, 알림 같은 상태 변경 전파에 사용한다. |
| gRPC | 초기 범위에서 제외한다. 내부 고성능 동기 호출이 검증되면 별도 ADR로 도입한다. |

정상 구매 1차 구현에서는 다음 흐름을 유지한다.

```text
Frontend -> REST -> catalog-service
Frontend -> REST -> order-service
Frontend -> REST -> payment-service
payment-service -> Kafka -> order-service
order-service -> Kafka -> notification-service
Frontend -> REST -> notification-service
```

## 6. gRPC를 바로 쓰지 않는 이유

- 현재 계약과 테스트 구조는 REST/OpenAPI 중심이다.
- Python, Go, TypeScript가 섞이면 proto와 codegen 관리 비용이 커진다.
- 정상 구매에서는 동기 내부 RPC보다 Kafka event의 느슨한 결합이 더 안전하다.
- gRPC는 admission, coupon usage reserve처럼 짧고 빈번한 내부 호출이 실제 병목일 때 도입한다.

## 7. 성능 검증 지표

| 영역 | 지표 | 초기 기준 |
| --- | --- | --- |
| 주문 생성 | admitted `POST /orders` p95 | local target `< 500ms` |
| 재고 정합성 | `oversell_count` | 항상 `0` |
| idempotency | `idempotency_replay_total`, `idempotency_conflict_total` | 재시도와 충돌이 구분되어야 한다. |
| outbox | `outbox_oldest_pending_age_seconds` | p95 `< 30s` |
| Kafka | `kafka_consumer_lag` | spike 후 `< 5m` 내 해소 |
| 결제 이벤트 | `payment_event_handler_failures_total` | 정상 구매에서 증가하지 않는다. |

## 8. 결정 규칙

- 언어 변경은 API path, event topic, DB 소유권을 바꾸지 않는다.
- 성능 이슈가 생기면 먼저 query, index, cache, HPA, connection pool을 확인한다.
- 그래도 병목이 남으면 해당 서비스 또는 worker만 Go로 전환한다.
- 여러 언어를 동시에 늘리지 않는다. 초기 운영 언어는 Python과 Go를 넘기지 않는다.
