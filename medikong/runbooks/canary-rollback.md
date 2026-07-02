# 운영 절차서: Canary rollback 대응

## 증상

- Argo Rollouts 분석 실패
- canary 오류율 증가
- canary p95/p99 증가
- outbox lag 또는 DLQ 증가
- `oversell_count > 0`

## 즉시 확인

1. Rollout dashboard에서 stable/canary traffic weight를 확인한다.
2. Prometheus 분석 metric 실패 항목을 확인한다.
3. canary pod log와 trace를 확인한다.
4. DB migration이 하위 호환되는지 확인한다.

## 대응

| 상황 | 조치 |
| --- | --- |
| 분석 실패 및 자동 rollback 시작 | rollback 완료까지 관찰한다. |
| 자동 rollback 미시작 | Argo Rollouts에서 수동 중단 또는 rollback을 수행한다. |
| oversell 감지 | 즉시 판매 중지, incident 선언, 데이터 정합성 조사 |
| event schema 불일치 | producer rollout 중지, consumer 호환성 확인 |

## 복구 후

- 실패한 canary version과 git SHA를 기록한다.
- 분석 threshold가 너무 느슨하거나 엄격했는지 확인한다.
- 재현 테스트를 `11-test-release-plan.md`에 추가한다.
