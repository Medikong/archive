# Backpressure Effectiveness Validation

## 목적

대용량 JSON 처리나 무거운 API 요청이 몰릴 때 request admission queue와 decode worker pool이 OOM 위험, 처리량, p95/p99 latency를 얼마나 안정화하는지 검증한다.

## 검증 질문

- handler가 body를 읽기 전에 admission slot을 얻도록 하면 max in-flight와 peak memory가 줄어드는가?
- gzip JSON decode 직전에 worker pool을 두면 decode 단계의 순간 메모리 피크가 줄어드는가?
- backpressure를 걸었을 때 처리량은 얼마나 줄고, p95/p99는 어느 지점에서 안정되는가?
- 사용자 경험 관점에서 `429` 없이 기다리게 할 수 있는 queue size와 max wait는 어느 정도인가?

## 비교 대상

| 구분 | 설명 |
| --- | --- |
| Baseline | 요청이 들어오면 바로 body read, gzip decode, JSON unmarshal, business logic을 수행한다. |
| Admission queue | body read 전에 route별 slot을 얻고, bounded queue에서 짧게 대기한다. |
| Decode worker pool | admission을 통과한 뒤 gzip/JSON decode 동시성을 별도로 제한한다. |
| Combined | admission queue와 decode worker pool을 함께 적용한다. |

## 실험 조건 초안

| 항목 | 후보 |
| --- | --- |
| payload size | p50, p95, max allowed JSON/gzip body |
| clients | 예상 피크, 피크 x2, 피크 x3 |
| admission slots | 4, 8, 16, 32 |
| queue size | slots x2, x4, x8 |
| max wait | 500ms, 1s, 2s |
| decode workers | 2, 4, 8, 16 |
| memory limit | 실제 배포 후보 limit의 70%, 100% |

## 측정 지표

| 지표 | 목적 |
| --- | --- |
| `admission_in_flight{route}` | 실제 처리 중 요청 수가 slot 근처로 제한되는지 확인 |
| `admission_queue_depth{route}` | 대기열 포화 여부 확인 |
| `admission_wait_seconds_bucket{route}` | 사용자가 기다린 시간 p95/p99 확인 |
| `admission_rejected_total{route,reason}` | queue/full 또는 max-wait 초과 비율 확인 |
| `json_decode_duration_seconds_bucket{route}` | decode 단계 지연 확인 |
| `request_body_compressed_bytes_bucket{route}` | 압축 body 크기 분포 확인 |
| `http_server_duration_seconds_bucket{route,status}` | 전체 API p95/p99 확인 |
| `go_memstats_heap_alloc_bytes` | Go heap 피크 확인 |
| `container_memory_working_set_bytes` | Kubernetes memory limit 대비 여유 확인 |
| `go_gc_duration_seconds` | GC pause 악화 여부 확인 |

## 성공 기준 초안

- peak memory가 baseline 대비 30% 이상 감소한다.
- `container_memory_working_set_bytes`가 memory limit의 70~80% 안에서 유지된다.
- success rate가 99% 이상이다. 단 의도적으로 overload를 검증하는 구간은 별도 표기한다.
- `admission_rejected_total`은 정상 피크 조건에서 1% 이하로 유지한다.
- p95/p99가 SLO 안에 들어오거나, baseline보다 악화되지 않는 균형점을 찾는다.

## 보관 산출물

```text
backpressure-effectiveness/
  README.md
  runs/
    <run-id>/
      execution.yaml
      k6-summary.json
      prometheus-query-results.json
      grafana-screenshots/
      report.md
```

## 선행 구현 후보

- route별 admission middleware
- decode worker pool 또는 bounded decoder wrapper
- k6 scenario: baseline, admission-only, decode-pool-only, combined
- Grafana dashboard: memory, p95/p99, admission queue, decode duration
- Prometheus recording rule 또는 dashboard query 정리

## 다음 액션

1. 실제로 무거운 route 후보를 정한다.
2. request body 크기와 decode record 수를 측정하는 metric을 먼저 붙인다.
3. baseline loadtest를 한 번 돌려 OOM 전조 지표와 현재 p95/p99를 잡는다.
4. admission slots와 queue size matrix를 실행한다.
5. 결과를 `evidence/loadtest/` 또는 이 폴더의 `runs/`에 연결한다.
