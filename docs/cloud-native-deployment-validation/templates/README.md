# Templates

이 폴더의 템플릿은 서비스별 배포 전략 실험을 시작할 때 복사해서 사용한다.

## 사용 순서

| 순서 | 템플릿 | 목적 |
| --- | --- | --- |
| 1 | [service-profile-template.md](service-profile-template.md) | 서비스 상태, DB, 정합성, 외부 연동, 롤백 가능성 정리 |
| 2 | [deployment-experiment-plan-template.md](deployment-experiment-plan-template.md) | 전략별 실험 계획, 성공 기준, 중단 기준 정리 |
| 3 | [observability-metrics-template.md](observability-metrics-template.md) | LGTM 기준 metric/log/trace/dashboard 설계 |
| 4 | [db-migration-strategy-template.md](db-migration-strategy-template.md) | DB 변경이 있는 경우 migration 정책 정리 |
| 5 | [chaos-scenario-template.md](chaos-scenario-template.md) | Chaos Mesh 장애 주입 계획 정리 |
| 6 | [result-analysis-report-template.md](result-analysis-report-template.md) | 실험 결과 수치와 해석 정리 |
| 7 | [deployment-decision-record-template.md](deployment-decision-record-template.md) | 서비스별 최종 배포 전략 결정 기록 |

## 작성 규칙

- 템플릿 원본은 결과값으로 덮어쓰지 않는다.
- 실험 ID, 서비스명, 환경, 담당자, 관련 실험 같은 기본 정보는 본문 표가 아니라 frontmatter에서 관리한다.
- 빈칸은 `TBD`로 두되, 수집할 지표의 단위와 출처는 남긴다.
- 서비스별 결과 문서는 실험 날짜와 서비스명을 파일명에 포함한다.
