---
title: DB 마이그레이션 전략 템플릿
type: db-migration-strategy-template
status: template
service_name: TBD
db_type: TBD
migration_id: TBD
related_deployment_experiment: TBD
migration_owner: TBD
created: TBD
updated: TBD
---

# DB 마이그레이션 전략 템플릿

## 변경 요약

- 변경 목적: TBD
- 변경 대상 table/collection: TBD
- 추가되는 schema: TBD
- 제거 또는 rename 대상: TBD
- 데이터 backfill 필요 여부: TBD

## 호환성 점검

| 질문 | 답변 | 조치 |
| --- | --- | --- |
| 구버전 코드가 새 schema에서도 동작하는가? | TBD | TBD |
| 신버전 코드가 기존 schema에서도 동작하는가? | TBD | TBD |
| rolling/canary 중 구버전과 신버전이 동시에 write 가능한가? | TBD | TBD |
| rollback 후 데이터 손실 가능성이 있는가? | TBD | TBD |
| contract 단계가 별도 배포로 분리되어 있는가? | TBD | TBD |

## Expand-Migrate-Contract 계획

| 단계 | 작업 | 배포 단위 | rollback 가능성 | 검증 지표 |
| --- | --- | --- | --- | --- |
| Expand | TBD | migration only | TBD | TBD |
| Migrate/backfill | TBD | job/batch | TBD | TBD |
| Read switch | TBD | app deploy/flag | TBD | TBD |
| Contract | TBD | migration only | 낮음 | TBD |

## 데이터 검증

| 검증 | 쿼리 또는 방법 | 성공 기준 |
| --- | --- | --- |
| row count 비교 | TBD | TBD |
| null/invalid 값 확인 | TBD | TBD |
| dual write diff | TBD | TBD |
| read path diff | TBD | TBD |
| backfill progress | TBD | TBD |

## 실패 대응

| 실패 조건 | 대응 | 담당자 |
| --- | --- | --- |
| migration lock 지연 | TBD | TBD |
| backfill 실패 | TBD | TBD |
| dual write 불일치 | TBD | TBD |
| read switch 후 오류 증가 | TBD | TBD |
| rollback 불가 상태 진입 | incident 선언 및 forward fix | TBD |
