# Archive

Medikong 문서와 증거 자료를 장기 보관하는 아카이빙 레포다.

이 레포는 현재 구현 repo가 아니라, 과거 설계 결정, 실행 증거, 트러블슈팅 기록, 발표/분석용 보조 자료를 다시 찾기 쉽게 보관하는 공간이다. 실제 코드 변경은 `services`, 배포 선언은 `gitops`, 인프라 구성은 `infra`, 작업공간 조율 문서는 `workspaces`를 기준으로 본다.

## 빠른 인덱스

| 영역 | 용도 | 시작 문서 |
| --- | --- | --- |
| `medikong/` | DropMong 전환 설계 패킷, ADR, 다이어그램, 운영 초안 | [medikong/README.md](medikong/README.md) |
| `medikong/adr/` | 구조적 의사결정 기록 | [medikong/adr/README.md](medikong/adr/README.md) |
| `architecture/` | 기존 Medikong 아키텍처, repo 경계, 관측성, 배포, 합성 E2E 설계 | [architecture/README.md](architecture/README.md) |
| `evidence/` | CI, 부하테스트, 보안, 관측성, 트래픽 검증 결과 | [evidence/README.md](evidence/README.md) |
| `research/` | 외부 기업 사례, 공개 기술 문서, 제품 설계 참고 자료 | [research/README.md](research/README.md) |
| `docs/` | 학습/실험/운영 검증 계획과 템플릿 | [docs/README.md](docs/README.md) |
| `validations/` | 실무 프로젝트에서 앞으로 검증할 운영 안정성 과제와 성공 기준 | [validations/README.md](validations/README.md) |
| `runbooks/` | 배포, 관측성, 운영 확인 절차 | [runbooks/README.md](runbooks/README.md) |
| `trouble/` | 장애, 실패, 운영 리스크 분석 기록 | [trouble/README.md](trouble/README.md) |
| `issues/` | GitHub Issue/Project 발행 전 작업 후보와 템플릿 | [issues/README.md](issues/README.md) |

## 폴더 구조

```text
archive/
  README.md
  GOAL.md
  architecture/
    README.md
    audit-logs/
    deployment/
    observability/
    synthetic-e2e/
  docs/
    README.md
    cloud-native-deployment-validation/
      README.md
      templates/
  evidence/
    README.md
    ci/
    loadtest/
    observability/
    resilience/
    security/
    traffic/
  issues/
    README.md
    template/
  medikong/
    README.md
    adr/
    assets/
    diagrams/
    runbooks/
  research/
    README.md
    auth-service-design/
      README.md
      companies/
      analysis/
      templates/
  validations/
    README.md
    backpressure-effectiveness/
      README.md
    oom-incident-snapshot-capture/
      README.md
  runbooks/
    README.md
    deployment/
    observability/
  trouble/
    README.md
    assets/
    templates/
    ecr-registry-403/
    image-multi-arch-pull-failure/
    trivy-pr-comment-gate/
```

## 주요 문서 묶음

### DropMong 전환 설계

| 문서 | 용도 |
| --- | --- |
| [medikong/00-product-scope.md](medikong/00-product-scope.md) | 제품 범위와 제외 범위 |
| [medikong/01-domain-model.md](medikong/01-domain-model.md) | 도메인 용어, 상태, 불변 조건 |
| [medikong/02-system-architecture.md](medikong/02-system-architecture.md) | 전체 시스템 구조와 repo 영향 |
| [medikong/03-service-boundaries.md](medikong/03-service-boundaries.md) | 서비스 책임 경계 |
| [medikong/04-data-design.md](medikong/04-data-design.md) | DB, transaction, idempotency, outbox 설계 |
| [medikong/05-api-contracts.md](medikong/05-api-contracts.md) | HTTP API와 에러 계약 |
| [medikong/06-event-contracts.md](medikong/06-event-contracts.md) | Kafka topic, event envelope, DLQ 규칙 |
| [medikong/07-critical-flows.md](medikong/07-critical-flows.md) | 주문, 결제, 만료, 장애, rollback 처리 |
| [medikong/08-infra-deployment.md](medikong/08-infra-deployment.md) | Istio, GitOps, HPA, KEDA, Rollouts |
| [medikong/09-observability-slo.md](medikong/09-observability-slo.md) | SLO, metric, log, trace, alert |
| [medikong/10-security.md](medikong/10-security.md) | 인증, 인가, mTLS, secret, audit |
| [medikong/11-test-release-plan.md](medikong/11-test-release-plan.md) | 테스트와 릴리즈 게이트 |
| [medikong/12-user-flows.md](medikong/12-user-flows.md) | 고객, 운영자, 플랫폼 운영자 사용자 흐름 |

### 운영 증거와 문제 기록

| 문서 | 용도 |
| --- | --- |
| [evidence/loadtest/README.md](evidence/loadtest/README.md) | 부하테스트 실행 조건과 결과 |
| [evidence/observability/README.md](evidence/observability/README.md) | 로그, 메트릭, 트레이스 검증 결과 |
| [evidence/security/README.md](evidence/security/README.md) | RBAC, ServiceAccount, NetworkPolicy 검증 결과 |
| [evidence/traffic/README.md](evidence/traffic/README.md) | canary, rollback, traffic policy 검증 |
| [trouble/README.md](trouble/README.md) | 트러블슈팅 기록 전체 인덱스 |

### 외부 사례 조사

| 문서 | 용도 |
| --- | --- |
| [research/auth-service-design/README.md](research/auth-service-design/README.md) | 국내/해외 기업 인증 서비스 설계 사례 조사 |

### 운영 절차

| 문서 | 용도 |
| --- | --- |
| [runbooks/deployment/tag-based-image-deploy.md](runbooks/deployment/tag-based-image-deploy.md) | 태그 기반 이미지 배포 절차 |
| [runbooks/deployment/ecr-kubelet-credential-provider.md](runbooks/deployment/ecr-kubelet-credential-provider.md) | ECR kubelet credential provider 도입 절차 |
| [runbooks/observability/aws-grafana-tunnel.md](runbooks/observability/aws-grafana-tunnel.md) | AWS Grafana SSH 터널 접속 |
| [runbooks/observability/synthetic-traffic-verification.md](runbooks/observability/synthetic-traffic-verification.md) | synthetic traffic 검증 |

## 작성 기준

- 루트 README는 큰 분류와 진입점만 다룬다.
- 세부 인덱스는 각 하위 폴더의 `README.md`에 둔다.
- 새 문서 묶음을 추가하면 루트 `빠른 인덱스`와 `폴더 구조`를 함께 갱신한다.
- 실행 로그, 캡처, 결과 파일은 주제별 하위 폴더에 두고, README에서는 읽는 순서와 대표 문서만 연결한다.
