---
title: 카오스 시나리오 템플릿
type: chaos-scenario-template
status: template
scenario_id: CHAOS-YYYYMMDD-SERVICE-N
service_name: TBD
related_deployment_experiment: TBD
chaos_mesh_resource: TBD
environment: TBD
blast_radius: TBD
owner: TBD
created: TBD
updated: TBD
---

# 카오스 시나리오 템플릿

## 실험 가설

- 주입할 장애: TBD
- 기대하는 시스템 반응: TBD
- 사용자에게 허용되는 영향: TBD
- 관측할 지표: TBD

## 영향 범위 제한

| 항목 | 값 |
| --- | --- |
| namespace | TBD |
| selector | TBD |
| duration | TBD |
| target version | stable / candidate / both |
| 제외 대상 | payment sandbox 외 실결제, production 외부 발송 등 |

## 실행 전 확인

| 확인 항목 | 상태 |
| --- | --- |
| rollback 담당자와 명령 확인 | TBD |
| Grafana annotation 준비 | TBD |
| synthetic test 준비 | TBD |
| 알림 수신 채널 확인 | TBD |
| Chaos Mesh 권한과 scope 확인 | TBD |

## 실행 기록

| 시각 | 작업 | 결과 |
| --- | --- | --- |
| TBD | Chaos experiment apply | TBD |
| TBD | 지표 확인 | TBD |
| TBD | experiment 종료 | TBD |
| TBD | 원복 확인 | TBD |

## 예상 결과와 실제 결과

| 항목 | 예상 | 실제 | 해석 |
| --- | --- | --- | --- |
| p99 | TBD | TBD | TBD |
| 5xx | TBD | TBD | TBD |
| retry/fallback | TBD | TBD | TBD |
| pod recovery | TBD | TBD | TBD |
| business metric | TBD | TBD | TBD |

## 종료 조건

- 실험 duration 만료: TBD
- 실패 기준 초과 시 즉시 종료 조건: TBD
- 실험 후 원복 확인 방법: TBD
