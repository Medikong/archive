# OOM Incident Snapshot Capture Validation

## 목적

부하테스트로 의도적으로 OOM 또는 OOM 위험 상태를 만들고, alert가 발생했을 때 당시의 Grafana screenshot, Prometheus query 결과, Kubernetes event/log가 자동으로 보관되는지 검증한다.

## 검증 질문

- OOMKilled 또는 memory pressure alert가 발생했을 때 collector가 자동으로 실행되는가?
- alert 시점 전후 5~10분의 Grafana dashboard screenshot을 저장할 수 있는가?
- screenshot뿐 아니라 Prometheus `query_range` 결과도 함께 저장되는가?
- Kubernetes event, pod describe, logs tail이 같은 run folder에 묶이는가?
- alert storm 상황에서 중복 캡처와 저장소 폭증을 막을 수 있는가?

## 비교 대상

| 구분 | 설명 |
| --- | --- |
| Manual capture | 장애 발생 후 사람이 Grafana와 kubectl로 수동 캡처한다. |
| Alert webhook capture | Alertmanager/Grafana alert webhook이 collector를 호출해 자동으로 캡처한다. |
| Kubernetes event watcher | OOMKilled event watcher가 collector를 호출한다. |

## 실험 조건 초안

| 항목 | 후보 |
| --- | --- |
| OOM 유도 방식 | memory leak endpoint, 대용량 JSON decode storm, 낮은 memory limit |
| alert trigger | memory working set > limit 80%, OOMKilled event, restart count 증가 |
| capture window | alert 기준 -10m ~ +5m |
| dashboard | service runtime, admission queue, loadtest execution, Kubernetes pod |
| storage | local PVC, S3/MinIO, gitignored artifact folder |
| duplicate control | alert fingerprint + cooldown |

## 캡처 산출물 구조

```text
oom-incident-snapshot-capture/
  runs/
    <incident-id>/
      alert.json
      summary.md
      grafana/
        memory.png
        latency.png
        admission.png
      prometheus/
        memory-query-range.json
        latency-query-range.json
        admission-query-range.json
      kubernetes/
        events.txt
        pod-describe.txt
        logs-tail.txt
```

## 필수 지표

| 지표 | 목적 |
| --- | --- |
| `container_memory_working_set_bytes` | OOM 전 memory pressure 확인 |
| `kube_pod_container_status_last_terminated_reason` | OOMKilled 여부 확인 |
| `kube_pod_container_status_restarts_total` | 재시작 발생 확인 |
| `go_memstats_heap_alloc_bytes` | Go heap 증가 확인 |
| `go_goroutines` | goroutine 폭증 여부 확인 |
| `http_server_duration_seconds_bucket` | latency spike 확인 |
| `admission_queue_depth` | backpressure 포화 여부 확인 |
| `admission_rejected_total` | overload 대응 여부 확인 |

## 성공 기준 초안

- OOMKilled 또는 OOM 위험 alert 발생 후 1분 안에 incident folder가 생성된다.
- `alert.json`, Grafana screenshot, Prometheus JSON, Kubernetes event/log가 같은 incident id 아래 저장된다.
- screenshot에는 alert 전후 시간 범위가 반영된다.
- Prometheus JSON만으로도 나중에 수치 재계산이 가능하다.
- 중복 alert는 cooldown 또는 fingerprint로 하나의 incident에 병합된다.

## 선행 구현 후보

- Alertmanager webhook receiver 또는 Grafana alert webhook receiver
- Grafana render API 또는 Playwright 기반 dashboard screenshot collector
- Prometheus `query_range` collector
- Kubernetes event/log collector
- incident artifact storage layout

## 다음 액션

1. Grafana dashboard UID와 panel 목록을 정한다.
2. Alertmanager webhook receiver를 작은 서비스로 만든다.
3. local 환경에서 낮은 memory limit으로 OOMKilled를 재현한다.
4. alert payload와 screenshot, query result, event/log가 같은 폴더에 남는지 검증한다.
5. 캡처 결과를 발표/회고에서 재사용 가능한 `summary.md`로 정리한다.
