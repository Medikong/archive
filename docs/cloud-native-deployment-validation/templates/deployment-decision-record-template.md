---
title: 배포 전략 결정 기록 템플릿
type: deployment-decision-record-template
status: template
decision_id: DDR-YYYYMMDD-SERVICE
service_name: TBD
decision_date: TBD
decision_owner: TBD
evidence_experiments:
  - TBD
decision_status: proposed
created: TBD
updated: TBD
---

# 배포 전략 결정 기록 템플릿

## 결정

이 서비스의 기본 배포 전략은 `TBD`로 한다.

## 배경

- 서비스 특성: TBD
- 주요 위험: TBD
- DB/migration 조건: TBD
- 외부 연동 조건: TBD
- 관측성 준비 상태: TBD

## 검토한 대안

| 대안 | 장점 | 단점 | 채택 여부 |
| --- | --- | --- | --- |
| Rolling | TBD | TBD | TBD |
| Canary | TBD | TBD | TBD |
| Blue/Green | TBD | TBD | TBD |
| Shadow | TBD | TBD | TBD |
| Feature Flag/A-B | TBD | TBD | TBD |

## 결정 근거

| 기준 | 근거 |
| --- | --- |
| 롤백 가능성 | TBD |
| DB 호환성 | TBD |
| 사용자 영향 최소화 | TBD |
| 운영 단순성 | TBD |
| 관측 가능성 | TBD |
| 비용/리소스 | TBD |

## 운영 규칙

- 기본 rollout 단계: TBD
- 자동 승격 조건: TBD
- 수동 승인 조건: TBD
- 즉시 rollback 조건: TBD
- DB contract 허용 조건: TBD
- feature flag 또는 kill switch: TBD

## 재검토 조건

- 서비스 트래픽 또는 중요도가 바뀌는 경우
- DB migration 방식이 바뀌는 경우
- 외부 연동이 추가되는 경우
- 관측 지표가 충분하지 않은 것으로 확인되는 경우
- 실험 결과가 기존 결정과 충돌하는 경우
