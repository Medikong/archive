# 운영 절차서: Kafka lag 대응

## 증상

- `kafka_consumer_lag` 증가
- notification 지연
- outbox pending age 증가
- DLQ depth 증가

## 즉시 확인

1. 어떤 topic과 consumer group에서 lag가 발생했는지 확인한다.
2. consumer pod 수와 KEDA ScaledObject 상태를 확인한다.
3. DLQ 증가 여부를 확인한다.
4. lag가 핵심 checkout을 막고 있는지 확인한다.

## 대응

| 상황 | 조치 |
| --- | --- |
| notification lag만 발생 | checkout 영향이 없으면 consumer 복구와 replay에 집중한다. |
| payment event lag | order confirmation 지연 가능성이 있으므로 page severity로 대응한다. |
| outbox relay 정지 | relay pod log와 DB pending row를 확인하고 relay restart를 검토한다. |
| poison event | DLQ로 격리하고 schema 불일치를 확인한다. |

## 복구 기준

- lag 해소 시간이 목표 이내로 회복된다.
- DLQ가 더 증가하지 않는다.
- duplicate event 처리 metric이 비정상적으로 증가하지 않는다.
