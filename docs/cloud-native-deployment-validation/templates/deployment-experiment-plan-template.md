---
title: 배포 실험 계획 템플릿
type: deployment-experiment-plan-template
status: template
experiment_id: DEPLOY-YYYYMMDD-SERVICE-STRATEGY
service_name: TBD
deployment_strategy: TBD
environment: TBD
image_tag: TBD
helm_values_change: TBD
argo_cd_application: TBD
owner: TBD
experiment_time: TBD
created: TBD
updated: TBD
---

# 배포 실험 계획 템플릿

## 실험 목적

- 검증하려는 가설: TBD
- 비교 기준 전략: TBD
- 이번 실험에서 결정할 것: TBD
- 이번 실험에서 결정하지 않을 것: TBD

## 사전 조건

| 조건 | 확인 방법 | 상태 |
| --- | --- | --- |
| 대상 service/deployment 상태 정상 | `kubectl`, Argo CD, Grafana | TBD |
| readiness/liveness probe 정상 | Kubernetes event, metric | TBD |
| rollback image tag 확인 | GitOps history, registry | TBD |
| 대시보드 준비 | Grafana link | TBD |
| synthetic test 준비 | Job 또는 CI link | TBD |
| DB migration 영향 확인 | migration 문서 | TBD |

## Istio 라우팅 계획

| 단계 | stable 비율 | candidate 비율 | 지속 시간 | 승격 조건 | 중단 조건 |
| --- | ---: | ---: | --- | --- | --- |
| baseline | 100% | 0% | TBD | baseline 지표 수집 | baseline 자체 이상 |
| step 1 | TBD | TBD | TBD | TBD | TBD |
| step 2 | TBD | TBD | TBD | TBD | TBD |
| final | 0% | 100% | TBD | synthetic 및 SLI 통과 | rollback |

## 성공 기준

| 지표 | 기준 | 수집 위치 |
| --- | --- | --- |
| 5xx rate | TBD | Prometheus/Grafana |
| p99 latency | TBD | Prometheus/Grafana |
| CPU/memory saturation | TBD | Prometheus/Grafana |
| pod restart | TBD | Kubernetes/Grafana |
| business failure | TBD | service metric |
| synthetic pass rate | TBD | synthetic Job/CI |

## 중단 및 롤백 기준

| 조건 | 조치 | 담당자 |
| --- | --- | --- |
| 5xx rate 기준 초과 | Istio route 이전 비율로 복귀 | TBD |
| p99 기준 초과 | traffic 확대 중단 | TBD |
| business failure 증가 | candidate 0% 전환 | TBD |
| migration 오류 | migration rollback 또는 forward fix 판단 | TBD |
| dashboard/metric 미수집 | 실험 중단 | TBD |

## 실행 기록

| 시각 | 작업 | 결과 | 링크 |
| --- | --- | --- | --- |
| TBD | TBD | TBD | TBD |
