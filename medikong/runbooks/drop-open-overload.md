# 운영 절차서: 드롭 오픈 과부하 대응

## 증상

- admitted 주문 latency 증가
- rejected 또는 queued RPS 급증
- DB connection 포화
- order pod CPU 포화

## 즉시 확인

1. 드롭 오픈 개요 dashboard에서 admitted, queued, rejected RPS를 확인한다.
2. `oversell_count`가 0인지 확인한다.
3. `order-service` p95/p99가 admitted traffic 기준인지 확인한다.
4. DB lock wait와 connection pool 상태를 확인한다.

## 대응

| 상황 | 조치 |
| --- | --- |
| rejected RPS만 증가, admitted latency 정상 | 정상적인 부하 차단일 수 있다. 공지와 dashboard를 유지한다. |
| admitted latency 증가 | admission 처리량을 낮추고 order HPA 상태를 확인한다. |
| DB lock wait 증가 | admission을 더 낮추고 order write path 배포를 동결한다. |
| oversell 감지 | 즉시 판매 중지, incident 선언, order/payment rollout rollback |

## 복구 후

- k6 profile과 실제 traffic 차이를 기록한다.
- admission threshold를 조정한다.
- outbox lag와 payment lag가 정상으로 돌아왔는지 확인한다.
