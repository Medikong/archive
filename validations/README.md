# Validation Backlog

KT TechUp 실무 프로젝트에서 검증할 운영 안정성 과제를 모아두는 공간이다.

이 폴더는 이미 실행된 결과를 보관하는 `evidence/`가 아니라, 앞으로 구현하고 검증할 항목의 목적, 실험 조건, 성공 기준, 산출물을 먼저 정의하는 backlog다. 검증이 끝나면 결과 캡처와 실행 로그는 각 항목 폴더 또는 `evidence/`의 대응 영역으로 연결한다.

## 검증 항목

| 항목 | 질문 | 시작 문서 |
| --- | --- | --- |
| Backpressure 도입 효과 | request admission queue, decode worker pool 같은 backpressure가 OOM 위험, p95, 처리량 안정화에 얼마나 기여하는가 | [backpressure-effectiveness/README.md](backpressure-effectiveness/README.md) |
| OOM incident snapshot 자동 기록 | 부하테스트로 OOM을 유도했을 때 Grafana screenshot, Prometheus query, Kubernetes event/log가 자동으로 보관되는가 | [oom-incident-snapshot-capture/README.md](oom-incident-snapshot-capture/README.md) |

## 공통 작성 기준

- 검증 항목마다 독립 폴더를 둔다.
- 각 폴더의 `README.md`에는 목적, 가설, 범위, 실행 조건, 측정 지표, 성공 기준, 보관 산출물을 적는다.
- 실행 전 계획과 실행 후 결과를 분리한다. 결과 파일이 커지면 `assets/` 또는 날짜별 run 폴더로 나눈다.
- 실제 서비스 repo, GitOps repo, Grafana dashboard, k6 script 경로가 정해지면 절대 경로 또는 repo-relative path를 함께 남긴다.
- OOM, 스파이크, 백프레셔 같은 판단은 추정으로 끝내지 않고 metric, log, trace, event 중 최소 두 가지 근거로 확인한다.
