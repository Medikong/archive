---
title: 배포 실험 결과 분석 템플릿
type: result-analysis-report-template
status: template
experiment_id: TBD
service_name: TBD
deployment_strategy: TBD
experiment_time: TBD
environment: TBD
image_tag: TBD
related_migration: TBD
related_chaos_scenario: TBD
created: TBD
updated: TBD
---

# 배포 실험 결과 분석 템플릿

## 한 줄 결론

TBD

## 실험 결과 요약

| 항목 | 값 | 판단 |
| --- | ---: | --- |
| 총 배포 시간 | TBD min | TBD |
| 첫 ready까지 시간 | TBD sec | TBD |
| 전체 ready까지 시간 | TBD sec | TBD |
| 롤백 시간 | TBD sec | TBD |
| synthetic 통과율 | TBD % | TBD |
| 최대 5xx rate | TBD % | TBD |
| p99 최대값 | TBD ms | TBD |
| CPU 최대값 | TBD % | TBD |
| memory 최대값 | TBD MiB | TBD |
| pod restart | TBD count | TBD |
| 업무 실패율 | TBD % | TBD |

## 전략별 비교

| 전략 | 배포 시간 | 롤백 시간 | p99 영향 | 오류율 영향 | 운영 난이도 | 종합 판단 |
| --- | ---: | ---: | ---: | ---: | --- | --- |
| Rolling | TBD | TBD | TBD | TBD | TBD | TBD |
| Canary | TBD | TBD | TBD | TBD | TBD | TBD |
| Blue/Green | TBD | TBD | TBD | TBD | TBD | TBD |
| Shadow | TBD | TBD | TBD | TBD | TBD | TBD |
| Feature Flag/A-B | TBD | TBD | TBD | TBD | TBD | TBD |

## 지표 분석

| 지표 | baseline | 실험 중 | 실험 후 | 해석 |
| --- | ---: | ---: | ---: | --- |
| request rate | TBD | TBD | TBD | TBD |
| 5xx rate | TBD | TBD | TBD | TBD |
| p95 latency | TBD | TBD | TBD | TBD |
| p99 latency | TBD | TBD | TBD | TBD |
| queue lag | TBD | TBD | TBD | TBD |
| DB latency | TBD | TBD | TBD | TBD |
| external dependency latency | TBD | TBD | TBD | TBD |
| business success rate | TBD | TBD | TBD | TBD |

## 로그/트레이스 분석

- 대표 trace: TBD
- 대표 error log query: TBD
- candidate에서만 발생한 오류: TBD
- stable/candidate 공통 오류: TBD
- 외부 연동 또는 DB 병목: TBD

## 카오스 실험 결과

| 시나리오 | 예상 결과 | 실제 결과 | 판단 |
| --- | --- | --- | --- |
| TBD | TBD | TBD | TBD |

## DB 마이그레이션 영향

| 항목 | 결과 |
| --- | --- |
| migration 단계 | TBD |
| rollback 가능 상태 | TBD |
| 데이터 불일치 여부 | TBD |
| backfill 소요 시간 | TBD |
| 추가 조치 | TBD |

## 최종 판단

- 이 서비스에 가장 적합한 전략: TBD
- 판단 근거: TBD
- 운영 전제: TBD
- 다음 실험: TBD
- 남은 위험: TBD
