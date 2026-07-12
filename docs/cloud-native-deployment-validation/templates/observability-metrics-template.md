---
title: 관측 지표 설계 템플릿
type: observability-metrics-template
status: template
service_name: TBD
experiment_id: TBD
dashboard_url: TBD
log_query_url: TBD
trace_query_url: TBD
alert_rule_url: TBD
created: TBD
updated: TBD
---

# 관측 지표 설계 템플릿

## Metric

| 지표 | PromQL 또는 출처 | 정상 기준 | 경고 기준 | 실패 기준 |
| --- | --- | --- | --- | --- |
| request rate | TBD | TBD | TBD | TBD |
| 5xx rate | TBD | TBD | TBD | TBD |
| p95 latency | TBD | TBD | TBD | TBD |
| p99 latency | TBD | TBD | TBD | TBD |
| CPU usage | TBD | TBD | TBD | TBD |
| memory usage | TBD | TBD | TBD | TBD |
| restart count | TBD | TBD | TBD | TBD |
| queue lag | TBD | TBD | TBD | TBD |
| DB connection usage | TBD | TBD | TBD | TBD |

## Log

| 필드 | 목적 | cardinality 주의 |
| --- | --- | --- |
| service_name | 서비스 필터 | 낮음 |
| version | stable/candidate 비교 | 낮음 |
| route | API별 오류 확인 | 중간 |
| status_code | 오류 분류 | 낮음 |
| error_code | 도메인 실패 분류 | 중간 |
| trace_id | trace 연결 | label 대신 structured field 권장 |
| request_id | 요청 추적 | label 대신 structured field 권장 |
| domain_id | 주문/쿠폰/드롭 추적 | label 대신 structured field 권장 |

## Trace

| span | 확인 항목 | 실패 해석 |
| --- | --- | --- |
| inbound HTTP | gateway/service latency | ingress 또는 app 처리 지연 |
| downstream HTTP/gRPC | 외부/내부 dependency latency | dependency 병목 |
| DB query | query latency, row count | migration/index 문제 |
| message publish | broker latency, failure | 비동기 처리 병목 |
| retry/fallback | retry count, fallback path | 장애 격리 동작 |

## Dashboard 패널

| 패널 | stable/candidate 비교 여부 | annotation 필요 여부 |
| --- | --- | --- |
| request/error/latency | 예 | 배포 단계 |
| resource saturation | 예 | 배포 단계 |
| pod health | 예 | 배포 단계 |
| business result | 예 | 배포 단계 |
| chaos experiment window | 예 | Chaos Mesh 시작/종료 |

## Alert 기준

| 알림 | 조건 | 실험 중 조치 |
| --- | --- | --- |
| candidate error spike | TBD | traffic 확대 중단 |
| latency regression | TBD | 이전 단계로 복귀 |
| pod instability | TBD | candidate 0% 전환 |
| business failure | TBD | 실험 중단 |
