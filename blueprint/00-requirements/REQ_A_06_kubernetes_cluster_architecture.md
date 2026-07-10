---
id: REQ.A.06
title: 쿠버네티스 클러스터 아키텍처 요구사항 정의
type: requirements
status: draft
tags: [requirements, dropmong, kubernetes, cluster, platform, sre, gitops, observability]
source: local
created: 2026-07-08
updated: 2026-07-08
---

# 쿠버네티스 클러스터 아키텍처 요구사항 정의

## 기본 정보

- Requirements ID: `REQ.A.06`
- 프로젝트: DropMong
- 작성 기준일: 2026-07-08
- 주요 사용자: 개발자, 플랫폼 운영자, 온콜 담당자, 보안/정책 담당자, 데이터/분석 담당자
- 주요 이해관계자: 구매자, 판매자, CS 담당자, 브랜드 운영자, 인프라 담당자, GitOps 담당자
- 설계 범위: Kubernetes 클러스터 구성 원칙, 인프라 환경별 클러스터 프로파일, namespace/워크로드 격리, Istio ingress/service mesh, DNS, image pull, Argo CD/Helm 기반 GitOps 배포, autoscaling, 보안 정책, secret/RBAC, 자체 운영 LGTM 관측성, 인시던트 대응, 네트워크 진단, 데이터/배치 워크로드, 운영 검증
- 제외 범위: 특정 클라우드 계정 생성, Terraform 리소스 상세, 실제 Helm values, 비용 산정 견적, 프로덕션 SLO 수치 확정, multi-region active-active 구현, 확정 설계 projection 문서 작성

## 구조 분석

### 요구사항 문서 구조

- 이 문서는 `REQ.A.01`의 드롭 커머스 피크 트래픽 요구와 `REQ.A.05`의 인증/세션 피크 요구를 받쳐 주는 플랫폼 요구사항이다.
- 기능 요구사항은 사용자가 직접 보는 화면 기능이 아니라, DropMong 운영자가 클러스터에서 반드시 제공해야 하는 운영 능력을 정의한다.
- 비기능 요구사항은 장애 격리, 관측성, 배포 안전성, 보안, 비용, 복구 가능성처럼 클러스터 설계가 만족해야 하는 기준을 정의한다.
- 후속 `50-service-design`, `80-sequence`, GitOps/infra 문서가 생기면 각 요구사항의 연결 문서를 갱신한다.
- 이번 문서는 아직 확정 설계가 아니므로 `archive/blueprint/00-requirements`의 요구사항으로만 둔다. 최종 설계 문서는 나중에 확정된 구조를 보여주는 projection 역할로 작성한다.
- 클러스터 구분은 `dev`, `qa`, `stage`, `prod` 같은 전형적인 환경명에 고정하지 않는다. 로컬 개발, private dev, 외부 제공 개발 환경, AWS 개발, AWS 프로덕션처럼 제공되는 인프라 조건에 따라 자유롭게 분류할 수 있어야 한다.
- 초기 목표 클러스터는 `local`, `private-dev`, `aws-dev`다. 관련 GitOps Helm chart 구성은 이미 존재하므로, 처음부터 새로 만드는 것이 아니라 기존 GitOps 구조를 개선하고 확장한다.

### 참고 자료 구조

- `work/tech-blog-library`는 기업 테크 블로그 수집본을 `posts/{site}/{year}/{published-date}-{slug}/index.md` 번들로 보관한다.
- 이번 문서는 수집본 중 Mercari, Bucketplace, Viva Republica(Toss), Oliveyoung의 Kubernetes, 플랫폼 운영, 관측성, 인시던트, 데이터 워크로드 신호를 우선 사용한다.
- `workspaces/docs/architecture`에는 DropMong의 현재 Kubernetes, 배포, 관측성, repo 책임 경계 문서가 이미 있다. 이 문서는 그 문서들을 구현 상세가 아니라 요구사항의 하위 근거로 연결한다.

### 현재 DropMong 연결 문서

| 문서 | 역할 |
| --- | --- |
| [workspaces Kubernetes architecture](../../../workspaces/docs/architecture/kubernetes-architecture/README.md) | namespace, Pod, container 단위 전체도 |
| [workspaces deployment](../../../workspaces/docs/architecture/deployment/README.md) | 이미지 태그, Helm values, Argo CD 반영 기준 |
| [workspaces observability](../../../workspaces/docs/architecture/observability/README.md) | metric, log, trace, dashboard 기준 |
| [workspaces synthetic E2E](../../../workspaces/docs/architecture/synthetic-e2e/README.md) | Kubernetes Job 기반 지속 검증 기준 |
| [workspaces repo boundaries](../../../workspaces/docs/architecture/repo-boundaries.md) | `service`, `gitops`, `infra`, `archive`, `workspaces` 책임 경계 |

## 문제 정의

- 사용자가 겪는 문제: 드롭 오픈 순간에는 홈/상세 조회, 로그인, 쿠폰, 재고 배정, 주문/결제, 알림이 동시에 몰린다. 클러스터의 DNS, ingress, image pull, autoscaling, queue, DB connection 중 하나가 흔들리면 사용자에게는 품절, 결제 실패, 인증 지연, 화면 오류가 한꺼번에 나타난다.
- 운영자가 겪는 문제: Kubernetes는 추상화가 많아 장애가 발생했을 때 Pod, Node, DNS, gateway, service mesh, app log, trace, queue, DB 중 어디서 시작됐는지 빠르게 좁히기 어렵다.
- 현재 방식의 한계: 단순히 "Kubernetes에 배포한다"는 말만으로는 운영 가능한 클러스터가 되지 않는다. 배포, 롤백, 정책 검증, secret 관리, 관측성, 네트워크 진단, workload 격리, 비용 관리, 인시던트 절차가 함께 요구사항으로 잡혀야 한다.
- 이 프로젝트가 해결해야 하는 것: DropMong은 작은 MVP라도 운영 가능한 Kubernetes 기본값을 가져야 한다. 핵심 서비스는 안전하게 배포되고, 피크 트래픽에서 병목을 관측할 수 있으며, 장애가 한 서비스에서 전체 서비스로 번지지 않게 격리되어야 한다. 이 과정은 학습/포트폴리오 목적까지 포함해 운영 깊이를 보여줄 수 있어야 한다.
- 해결하지 않는 것: 초기 요구사항은 대규모 엔터프라이즈 플랫폼 전체를 한 번에 만들지 않는다. multi-region, 완전 자동 remediation, 전사 플랫폼 포털은 후속 확장 대상으로 둔다.

## 도출 근거

| 회사/근거 | 확인된 문제 | 잘한 점 | DropMong에 반영할 개선점 |
| --- | --- | --- | --- |
| Mercari DNS | Kubernetes DNS query 폭증, NXDOMAIN, truncated response, cache refresh error가 production outage로 이어졌다. | NodeLocal DNSCache 도입 후 kube-dns 호출과 DNS error를 크게 줄이고, DNS를 클러스터 신뢰성의 1급 지표로 다뤘다. | DropMong도 kube-dns/CoreDNS saturation, DNS error, service discovery 실패를 별도 관측/알림 대상으로 둔다. |
| Mercari network diagnosis | 수백 개 microservice와 Istio mTLS 환경에서는 네트워크 문제를 일반 로그만으로 찾기 어렵다. | Ephemeral Container와 netshoot 기반 pod-level packet capture 절차를 self-service로 정리했다. | 운영 권한을 무제한 열지 않고, 임시 권한과 감사 로그가 있는 네트워크 진단 절차를 둔다. |
| Mercari OPA Gatekeeper | microservice가 늘수록 개발자에게 컨테이너 보안 책임을 모두 맡기기 어렵다. | hostNetwork, hostPath, privileged container, capability, registry 정책을 admission 단계에서 검증했다. | DropMong GitOps/cluster에는 기본 보안 정책을 code review와 admission policy로 이중 적용한다. |
| Mercari platform debt | 빠른 검증을 위해 core platform과 다른 stack을 택한 뒤 latency, UX, 복잡도, 비용 부채가 커졌다. | Cloud Run에서 unified GKE cluster로 점진 이전하고, Datadog/WarpSpeed CD로 관측 가능한 rollout을 만들었다. | MVP도 임시 예외를 만들 수 있지만, 예외가 표준 경로로 돌아오는 조건과 rollback 기준을 문서화한다. |
| Bucketplace Spark on Kubernetes | 매일 20,000개 이상 Spark 작업과 1,000개 이상 transient cluster 생성이 대기 시간, 비용, 모니터링 파편화를 만들었다. | Spark on Kubernetes, Karpenter, Datadog, 통합 Submitter, fallback plugin, gradual ramp-up을 적용했다. | 데이터/배치 워크로드는 앱 서비스와 분리하되 같은 클러스터 운영 언어, 관측성, 권한 체계로 다룬다. |
| Bucketplace model serving | 여러 모델이 같은 Pod에서 실행되고 config가 분산되어 모델별 확장, 롤백, 장애 원인 분석이 어려웠다. | Kubernetes Controller, MLflow Model Registry alias, 모델별 autoscaling/endpoint/dashboard를 만들었다. | 장기적으로 추천/검색/AI 서빙을 넣을 경우 모델별 Deployment, version, traffic, cost 관측을 요구한다. |
| Toss Spark Connect | long-running Spark Connect 서버 하나에 여러 세션이 붙으면 Driver가 단일 장애점이 된다. | gateway, session 보호, replication, consistent hashing 같은 운영 장치를 고민했다. | 공유 서버형 워크로드는 세션 격리, 장애 영향 범위, gateway routing, quota를 먼저 정의한다. |
| Oliveyoung incident/observability | 세일 피크에서 주문, 쿠폰, 인증, MQ, 배송 배정이 동시에 병목이 되고, 업무 영향 해석이 늦어질 수 있다. | Datadog dashboard, incident level, Slack 선언, 5 Why, 온콜 자동화, chaos test를 운영 체계로 묶었다. | DropMong은 Kubernetes 지표만 보지 않고 드롭별 구매 성공률, 쿠폰 발급, 결제 실패, queue lag를 같은 대시보드에서 본다. |

## 회사별 배울 점과 주의할 점

| 회사 | 배울 점 | 주의할 점 |
| --- | --- | --- |
| Mercari | DNS, image pull, network debug, platform self-service처럼 "기본 경로"를 제품처럼 관리한다. | GKE, Istio, 대규모 microservice 전제가 강하므로 DropMong MVP에 그대로 복제하면 운영면이 과해질 수 있다. |
| Bucketplace | 데이터/ML 워크로드도 Kubernetes의 autoscaling, dashboard, Git 기반 선언으로 표준화한다. | Spark/ML 플랫폼 요구는 초기 DropMong 핵심 구매 경로보다 후순위일 수 있다. |
| Toss | Spark Connect 같은 공유 서버형 워크로드에서 단일 장애점과 세션 보호를 먼저 다룬다. | 금융/데이터 인프라 맥락이 강하므로 일반 API 서버 요구사항과 구분해야 한다. |
| Oliveyoung | 피크 트래픽, 장애 선언, 업무 지표, 복구 API, chaos test를 운영 기준으로 연결한다. | ECS/Fargate, Datadog 등 특정 도구보다 운영 원칙을 가져와야 한다. |

## 페인포인트 개선 매핑

| 문제 ID | 페인포인트 | 개선 방향 | 연결 기능 요구사항 | 연결 비기능 요구사항 | 근거 |
| --- | --- | --- | --- | --- | --- |
| `REQ.A.06.PP-001` | Kubernetes에 올렸지만 namespace, service account, secret, network boundary가 섞여 장애 영향 범위가 커질 수 있다. | 서비스 도메인, 플랫폼, 데이터, 관측성, synthetic 실행을 namespace와 RBAC로 분리한다. | `REQ.A.06.FR-001`, `REQ.A.06.FR-002`, `REQ.A.06.FR-003` | `REQ.A.06.NFR-001`, `REQ.A.06.NFR-002` | [workspaces repo boundaries](../../../workspaces/docs/architecture/repo-boundaries.md) |
| `REQ.A.06.PP-002` | 피크 시간 DNS 장애가 service timeout과 cascading failure로 번질 수 있다. | CoreDNS/kube-dns, NodeLocal DNSCache 적용 여부, DNS error/latency 지표를 운영 기준에 넣는다. | `REQ.A.06.FR-006`, `REQ.A.06.FR-018` | `REQ.A.06.NFR-006`, `REQ.A.06.NFR-018` | [Mercari DNS](https://engineering.mercari.com/en/blog/entry/20250515-from-dns-failures-to-resilience-how-nodelocal-dnscache-saved-the-day/) |
| `REQ.A.06.PP-003` | image pull 실패가 배포 실패나 장애 복구 지연으로 이어질 수 있다. | registry mirror/cache, imagePullSecret, pull 실패 알림, staging 검증, break-glass 권한을 둔다. | `REQ.A.06.FR-007`, `REQ.A.06.FR-019` | `REQ.A.06.NFR-007`, `REQ.A.06.NFR-017` | [Mercari Artifact Registry](https://engineering.mercari.com/en/blog/entry/20250523-when-caching-hides-the-truth-a-vpc-service-controls-artifact-registry-tale/) |
| `REQ.A.06.PP-004` | 네트워크 장애 조사 절차가 없으면 온콜이 Pod/Node/mesh 중 어디를 볼지 늦게 결정한다. | Ephemeral Container 기반 pod-level packet capture와 권한 승인/감사 절차를 둔다. | `REQ.A.06.FR-020`, `REQ.A.06.FR-021` | `REQ.A.06.NFR-012`, `REQ.A.06.NFR-020` | [Mercari Packet Capture](https://engineering.mercari.com/en/blog/entry/20251218-capturing-network-packets-in-kubernetes/) |
| `REQ.A.06.PP-005` | Kubernetes 보안을 사람 리뷰에만 맡기면 hostPath, privileged, capability 같은 위험 설정이 들어갈 수 있다. | admission policy와 GitOps review로 금지 설정을 자동 검증한다. | `REQ.A.06.FR-008`, `REQ.A.06.FR-009` | `REQ.A.06.NFR-010`, `REQ.A.06.NFR-011` | [Mercari OPA Gatekeeper](https://engineering.mercari.com/en/blog/entry/20201222-enhance-kubernetes-security-with-opa-gatekeeper/) |
| `REQ.A.06.PP-006` | 배포가 성공해도 readiness, migration, rollback 기준이 없으면 사용자 요청이 끊길 수 있다. | rolling update, readiness/liveness, backward-compatible API, migration 분리, rollback 기준을 배포 요구사항으로 둔다. | `REQ.A.06.FR-010`, `REQ.A.06.FR-011` | `REQ.A.06.NFR-004`, `REQ.A.06.NFR-005`, `REQ.A.06.NFR-017` | [workspaces deployment](../../../workspaces/docs/architecture/deployment/README.md) |
| `REQ.A.06.PP-007` | 관측성이 Pod CPU/Memory에 머무르면 드롭 실패 원인을 설명하기 어렵다. | 시스템 지표, 서비스 SLI, 비즈니스 지표, trace/log를 같은 incident view로 연결한다. | `REQ.A.06.FR-012`, `REQ.A.06.FR-013`, `REQ.A.06.FR-014` | `REQ.A.06.NFR-013`, `REQ.A.06.NFR-014`, `REQ.A.06.NFR-015` | [Oliveyoung Datadog 운영](https://oliveyoung.tech/2024-08-05/dash-2024-slide/) |
| `REQ.A.06.PP-008` | queue, worker, 비동기 재처리 상태가 서비스 메트릭과 분리되면 사용자 성공/실패를 추적하기 어렵다. | queue lag, DLQ, 재처리 수, consumer error를 드롭/쿠폰/알림 지표와 함께 본다. | `REQ.A.06.FR-014`, `REQ.A.06.FR-018` | `REQ.A.06.NFR-014`, `REQ.A.06.NFR-016` | [Oliveyoung RabbitMQ](https://oliveyoung.tech/2025-10-28/coupon-mq-issue/) |
| `REQ.A.06.PP-009` | 데이터/배치 워크로드가 별도 운영 체계에 있으면 권한, 모니터링, 비용이 파편화된다. | batch/data namespace, resource quota, Karpenter/HPA, 작업 성공률, 비용 태그를 요구한다. | `REQ.A.06.FR-015`, `REQ.A.06.FR-016` | `REQ.A.06.NFR-008`, `REQ.A.06.NFR-021` | [Bucketplace Spark on Kubernetes](https://www.bucketplace.com/post/2025-05-23-%EC%98%A4%EB%8A%98%EC%9D%98%EC%A7%91-spark-on-kubernetes-%EB%8F%84%EC%9E%85-%EB%B0%8F-%EA%B0%9C%EC%84%A0-%EC%97%AC%EC%A0%95/) |
| `REQ.A.06.PP-010` | 모델/추천 서빙을 한 Pod나 한 config에 몰아넣으면 모델별 확장과 롤백이 어렵다. | 모델별 Deployment/Service, version alias, traffic, cost, rollback 지표를 후속 요구사항으로 남긴다. | `REQ.A.06.FR-017` | `REQ.A.06.NFR-009`, `REQ.A.06.NFR-021` | [Bucketplace 모델 서빙](https://www.bucketplace.com/post/2025-03-14-%EA%B0%9C%EC%9D%B8%ED%99%94-%EC%B6%94%EC%B2%9C-%EC%8B%9C%EC%8A%A4%ED%85%9C-3-%EB%AA%A8%EB%8D%B8-%EC%84%9C%EB%B9%99/) |
| `REQ.A.06.PP-011` | synthetic 검증이 사용자의 실제 경로와 연결되지 않으면 배포 후 핵심 여정 실패를 늦게 발견한다. | Kubernetes Job 기반 synthetic E2E를 배포 후 검증과 지속 모니터링에 연결한다. | `REQ.A.06.FR-022`, `REQ.A.06.FR-023` | `REQ.A.06.NFR-022` | [workspaces synthetic E2E](../../../workspaces/docs/architecture/synthetic-e2e/README.md) |
| `REQ.A.06.PP-012` | 인시던트 선언, 전파, 회고 기준이 없으면 장애가 사람 기억에 의존한다. | incident level, owner, channel, timeline, postmortem, action item을 요구사항으로 둔다. | `REQ.A.06.FR-024`, `REQ.A.06.FR-025` | `REQ.A.06.NFR-023`, `REQ.A.06.NFR-024` | [Oliveyoung incident](https://oliveyoung.tech/2024-01-23/incident/) |

## 목표

- 사용자 목표: 구매자는 피크 상황에서도 조회, 로그인, 구매 시도, 결제 결과 확인을 예측 가능한 응답과 메시지로 경험한다.
- 비즈니스 목표: DropMong은 드롭 오픈처럼 짧은 시간에 매출과 신뢰가 결정되는 이벤트를 안정적으로 운영한다.
- 운영 목표: 온콜 담당자는 gateway, app, DNS, node, queue, DB, 외부 연동 중 어디가 문제인지 빠르게 좁히고, 임시 차단/롤백/재처리를 수행한다.
- 기술 목표: 서비스 배포, secret, 네트워크, 관측성, 보안 정책, synthetic 검증은 Argo CD와 Helm 기반 GitOps, 코드 리뷰로 재현 가능하게 관리한다.
- 학습/포트폴리오 목표: `local`, `private-dev`, `aws-dev`를 시작점으로 실제 운영 환경이 어떻게 늘어나는지 보여주고, 기존 GitOps repo 구조를 발전시키는 과정을 남긴다.

## 사용자 유형

| Actor ID | 사용자 | 목표 | 주요 행동 |
| --- | --- | --- | --- |
| `ACTOR-DEVELOPER` | 개발자 | 서비스를 안전하게 배포하고 장애 원인을 추적한다. | manifest 변경, 배포 확인, 로그/trace 조회, synthetic 결과 확인 |
| `ACTOR-PLATFORM-OPERATOR` | 플랫폼 운영자 | 클러스터 기본 경로와 운영 정책을 관리한다. | namespace/RBAC/secret/policy 관리, DNS/ingress/observability 운영 |
| `ACTOR-ONCALL` | 온콜 담당자 | 피크 장애를 빠르게 선언하고 완화한다. | 알림 수신, 대시보드 확인, 롤백, packet capture, postmortem 작성 |
| `ACTOR-SECURITY` | 보안/정책 담당자 | 위험한 Kubernetes 설정과 secret 노출을 막는다. | admission policy 관리, 감사 로그 확인, 예외 승인 |
| `ACTOR-DATA` | 데이터/분석 담당자 | 배치/분석 워크로드를 안정적으로 실행한다. | batch job 제출, 작업 상태 확인, 비용/자원 사용량 확인 |

## 기능 요구사항

| Req ID | 요구사항 | 사용자 | 우선순위 | 연결 Page/UC |
| --- | --- | --- | --- | --- |
| `REQ.A.06.FR-001` | 시스템은 `local`, `private-dev`, `aws-dev`를 초기 클러스터 프로파일로 정의하고, 이후 외부 개발, QA, AWS 프로덕션처럼 제공 인프라 조건에 따라 새 프로파일을 추가할 수 있어야 한다. | 플랫폼 운영자 | Must | GitOps 예정 |
| `REQ.A.06.FR-002` | 시스템은 namespace별 service account, Role/RoleBinding, secret 접근 범위를 분리한다. | 플랫폼 운영자, 보안 담당자 | Must | GitOps 예정 |
| `REQ.A.06.FR-003` | 시스템은 구매/주문/결제/쿠폰/인증 같은 핵심 서비스가 서로 다른 Deployment와 Service로 배포될 수 있게 한다. | 개발자 | Must | SVC.A 예정 |
| `REQ.A.06.FR-004` | 시스템은 기존 Kong Ingress 기준을 Istio ingress gateway 기준으로 전환하고, 외부 HTTP 트래픽에 route별 timeout, retry, rate limit 정책을 적용할 수 있어야 한다. | 플랫폼 운영자 | Must | API.A 예정 |
| `REQ.A.06.FR-005` | 시스템은 Istio service mesh를 포함하고, 내부 서비스 간 호출에 대해 service discovery, timeout, circuit breaker, traffic policy, fallback 기준을 서비스별로 정의할 수 있어야 한다. | 개발자, 온콜 담당자 | Must | SVC.A 예정 |
| `REQ.A.06.FR-006` | 시스템은 CoreDNS/kube-dns 상태와 DNS query error, latency, saturation을 수집하고 알림으로 연결한다. | 온콜 담당자 | Must | 관측성 예정 |
| `REQ.A.06.FR-007` | 시스템은 image registry, imagePullSecret, image pull 실패, pull latency를 배포 검증과 알림 대상으로 관리한다. | 플랫폼 운영자 | Must | GitOps 예정 |
| `REQ.A.06.FR-008` | 시스템은 privileged container, hostNetwork, hostPath, 불필요한 Linux capability, 허용되지 않은 registry를 정책으로 제한한다. | 보안 담당자 | Must | 정책 예정 |
| `REQ.A.06.FR-009` | 시스템은 보안 정책 예외가 필요할 때 대상 namespace, workload, 기간, 승인자, 사유를 기록한다. | 보안 담당자, 플랫폼 운영자 | Must | 운영자 기능 예정 |
| `REQ.A.06.FR-010` | 시스템은 모든 서비스 배포에 readiness probe, liveness probe, resource request/limit, graceful shutdown 기준을 요구한다. | 개발자 | Must | GitOps 예정 |
| `REQ.A.06.FR-011` | 시스템은 rolling update, canary 또는 blue/green 중 하나 이상의 점진 배포와 빠른 rollback 경로를 제공한다. | 개발자, 온콜 담당자 | Must | 배포 문서 예정 |
| `REQ.A.06.FR-012` | 시스템은 Pod/Container/Node/Deployment 상태, gateway 지표, DNS 지표, queue 지표를 Prometheus/Grafana에서 조회할 수 있게 한다. | 온콜 담당자 | Must | [observability](../../../workspaces/docs/architecture/observability/README.md) |
| `REQ.A.06.FR-013` | 시스템은 서비스별 request latency, error rate, saturation, external dependency latency, domain result metric을 노출한다. | 개발자, 온콜 담당자 | Must | SVC.A 예정 |
| `REQ.A.06.FR-014` | 시스템은 trace_id, request_id, user_id, drop_id, order_id, coupon_id를 로그/trace/audit event에서 연결해 조회할 수 있게 한다. | CS, 온콜 담당자 | Must | 관측성 예정 |
| `REQ.A.06.FR-015` | 시스템은 batch/data workload를 서비스 API workload와 구분해 실행하고, resource quota와 schedule window를 적용할 수 있어야 한다. | 데이터/분석 담당자 | Should | 데이터 플랫폼 예정 |
| `REQ.A.06.FR-016` | 시스템은 Karpenter, HPA, VPA 또는 동등한 autoscaling 정책으로 API workload와 batch workload의 확장 기준을 분리한다. | 플랫폼 운영자 | Should | infra 예정 |
| `REQ.A.06.FR-017` | 시스템은 추천/검색/AI 모델 서빙이 도입될 경우 모델별 Deployment, version, endpoint, autoscaling, rollback, dashboard를 제공할 수 있어야 한다. | 데이터/분석 담당자, 개발자 | Could | ML 플랫폼 예정 |
| `REQ.A.06.FR-018` | 운영자는 드롭 이벤트 기간에 gateway traffic, service p95/p99, DNS error, pod restart, queue lag, DB connection, 결제/알림 실패율을 한 화면에서 확인한다. | 온콜 담당자 | Must | 운영 대시보드 예정 |
| `REQ.A.06.FR-019` | 운영자는 registry/image pull 장애 시 영향 workload, 실패 이벤트, 최근 배포 이력을 확인하고 rollback 또는 image 재배포를 수행할 수 있다. | 온콜 담당자 | Should | 배포 문서 예정 |
| `REQ.A.06.FR-020` | 운영자는 네트워크 장애 조사 시 대상 Pod에 임시 디버그 컨테이너를 붙여 제한된 시간 동안 packet capture를 수행할 수 있다. | 온콜 담당자 | Should | runbook 예정 |
| `REQ.A.06.FR-021` | 시스템은 임시 디버그 권한 사용 시 사용자, 대상 리소스, 명령 목적, 시작/종료 시각을 감사 로그로 남긴다. | 보안 담당자 | Must | 감사 로그 예정 |
| `REQ.A.06.FR-022` | 시스템은 Kubernetes Job 기반 synthetic E2E를 통해 홈 조회, 로그인, 드롭 상세, 구매 시도, 주문 상태 조회 같은 핵심 경로를 주기적으로 검증한다. | 온콜 담당자 | Must | [synthetic E2E](../../../workspaces/docs/architecture/synthetic-e2e/README.md) |
| `REQ.A.06.FR-023` | 배포 파이프라인은 배포 후 synthetic E2E와 핵심 서비스 metric을 확인하고 실패 시 수동 승인 없이 다음 단계로 넘어가지 않게 할 수 있어야 한다. | 개발자, 플랫폼 운영자 | Should | CI/CD 예정 |
| `REQ.A.06.FR-024` | 시스템은 incident level, owner, 영향 범위, 완화 조치, 고객 영향, postmortem 링크를 남기는 인시던트 기록 경로를 제공한다. | 온콜 담당자 | Must | 운영자 기능 예정 |
| `REQ.A.06.FR-025` | 운영자는 피크 중 특정 드롭, API route, 기능 flag, background worker를 배포 없이 일시 제한하거나 읽기 전용으로 전환할 수 있어야 한다. | 온콜 담당자, 플랫폼 운영자 | Must | [REQ.A.01](./REQ_A_01_limited_drop_commerce.md) |
| `REQ.A.06.FR-026` | 시스템은 Helm chart와 environment-specific values를 Argo CD가 동기화하는 GitOps 구조를 기본 배포 경로로 사용한다. Kustomize values는 기본 경로로 사용하지 않는다. | 플랫폼 운영자, 개발자 | Must | GitOps 예정 |
| `REQ.A.06.FR-027` | 시스템은 Loki, Grafana, Tempo, Mimir 또는 Prometheus 기반 LGTM 관측성 스택을 자체 운영 기준으로 둔다. Datadog 같은 외부 SaaS는 비용 문제로 기본 후보에서 제외한다. | 플랫폼 운영자, 온콜 담당자 | Must | 관측성 예정 |

## 비기능 요구사항

| Req ID | 요구사항 | 기준 | 연결 문서 |
| --- | --- | --- | --- |
| `REQ.A.06.NFR-001` | namespace와 RBAC는 최소 권한 원칙을 따라야 한다. | 서비스별 service account는 자기 namespace와 필요한 secret만 접근한다. | GitOps 예정 |
| `REQ.A.06.NFR-002` | 클러스터 책임 경계는 repo별로 분리되어야 한다. | 서비스 코드는 manifest 기본값을 알 수 있지만 클러스터 bootstrap과 Argo CD Application은 `infra`/`gitops` 책임으로 둔다. | [repo boundaries](../../../workspaces/docs/architecture/repo-boundaries.md) |
| `REQ.A.06.NFR-003` | 외부 ingress와 내부 service mesh 정책은 명확히 분리되어야 한다. | 외부 공개 API는 Istio ingress gateway를 통과하고, 내부 호출은 service DNS와 Istio traffic policy를 따른다. | API.A 예정 |
| `REQ.A.06.NFR-004` | 배포 중 사용자 요청이 중단되지 않아야 한다. | readiness가 통과하지 않은 Pod는 트래픽을 받지 않고, graceful shutdown은 진행 중 요청을 끊지 않는다. | 배포 문서 예정 |
| `REQ.A.06.NFR-005` | database migration과 API 배포는 역호환성을 가져야 한다. | breaking schema change는 expand-migrate-contract 또는 동등한 절차 없이 배포하지 않는다. | persistence 예정 |
| `REQ.A.06.NFR-006` | DNS는 클러스터 핵심 의존성으로 관측해야 한다. | DNS latency, error rate, kube-dns/CoreDNS saturation, NodeLocal DNSCache 적용 여부를 대시보드에 둔다. | 관측성 예정 |
| `REQ.A.06.NFR-007` | image pull 경로는 장애 원인을 추적할 수 있어야 한다. | image pull backoff, registry auth failure, cache/mirror miss, pull latency가 이벤트와 로그로 확인된다. | GitOps 예정 |
| `REQ.A.06.NFR-008` | autoscaling은 워크로드 성격별로 분리해야 한다. | API, worker, scheduler, batch, data workload가 같은 기준으로 node/pod를 무제한 점유하지 않는다. | infra 예정 |
| `REQ.A.06.NFR-009` | 공유 서버형 워크로드는 단일 장애점과 세션 격리를 명시해야 한다. | Spark Connect, model serving, long-running worker는 세션/tenant별 영향 범위를 정의한다. | 데이터 플랫폼 예정 |
| `REQ.A.06.NFR-010` | 위험한 Kubernetes 설정은 admission 단계에서 차단해야 한다. | privileged, hostNetwork, hostPath, unsafe capabilities, unapproved registry는 기본 차단된다. | 정책 예정 |
| `REQ.A.06.NFR-011` | 정책 예외는 감사 가능해야 한다. | 예외에는 기간, 승인자, 사유, owner, 제거 예정일이 있어야 한다. | 감사 로그 예정 |
| `REQ.A.06.NFR-012` | 운영 진단 도구는 권한과 감사 로그를 동반해야 한다. | packet capture, debug shell, secret 조회, pod exec는 대상과 사용자를 남긴다. | runbook 예정 |
| `REQ.A.06.NFR-013` | 시스템 메트릭과 서비스 메트릭은 역할을 나눠야 한다. | Pod/Node 상태는 Kubernetes/exporter가, 사용자 요청/도메인 결과는 서비스 코드가 노출한다. | [observability](../../../workspaces/docs/architecture/observability/README.md) |
| `REQ.A.06.NFR-014` | 업무 지표는 인프라 지표와 연결되어야 한다. | 드롭별 구매 시도, 재고 배정, 결제 실패, 쿠폰 발급, 알림 실패가 같은 시간축에서 보인다. | [REQ.A.01](./REQ_A_01_limited_drop_commerce.md) |
| `REQ.A.06.NFR-015` | 로그, metric, trace는 공통 correlation id를 가져야 한다. | `trace_id`, `request_id`, `service_name`, 주요 domain id가 낮은 cardinality 정책에 맞춰 전파된다. | 관측성 예정 |
| `REQ.A.06.NFR-016` | queue와 worker는 lag와 실패 상태를 운영자가 이해할 수 있어야 한다. | queue length, consumer lag, DLQ, retry count, processing latency가 dashboard와 alert에 있다. | worker 예정 |
| `REQ.A.06.NFR-017` | 롤백은 배포 전략의 일부여야 한다. | image tag, GitOps values, migration, feature flag, route policy별 rollback 절차가 문서화되어 있다. | 배포 문서 예정 |
| `REQ.A.06.NFR-018` | 장애 격리는 서비스 호출 정책에 반영되어야 한다. | timeout, retry budget, circuit breaker, bulkhead, fallback이 외부 연동과 내부 호출에 적용된다. | SVC.A 예정 |
| `REQ.A.06.NFR-019` | secret은 평문 values나 문서에 남기지 않아야 한다. | Kubernetes Secret, sealed secret, external secret 중 결정된 경로만 사용하고 조회 이력을 남긴다. | GitOps 예정 |
| `REQ.A.06.NFR-020` | 네트워크 진단은 피크 중에도 실행 가능해야 한다. | 온콜이 사전 승인된 runbook으로 10분 안에 대상 Pod capture 또는 대체 진단을 시작할 수 있다. | runbook 예정 |
| `REQ.A.06.NFR-021` | 비용은 namespace, service, workload 기준으로 추적 가능해야 한다. | resource request/limit, node group, batch job, model serving traffic/cost를 owner별로 볼 수 있다. | FinOps 예정 |
| `REQ.A.06.NFR-022` | synthetic E2E는 실제 사용자 경로와 운영 조회를 연결해야 한다. | 실패한 synthetic run은 Kubernetes Job 상태, Loki log, Tempo trace, 서비스 metric으로 추적된다. | [synthetic E2E](../../../workspaces/docs/architecture/synthetic-e2e/README.md) |
| `REQ.A.06.NFR-023` | 인시던트 기록은 재발 방지 작업으로 이어져야 한다. | postmortem에는 timeline, root cause, customer impact, action item, owner, due date가 있다. | 운영자 기능 예정 |
| `REQ.A.06.NFR-024` | 클러스터 운영 문서는 실제 runbook으로 사용할 수 있어야 한다. | 장애 유형별 첫 확인 명령, 대시보드, rollback, 연락 경로가 문서에 있다. | runbook 예정 |
| `REQ.A.06.NFR-025` | 환경 구분은 이름보다 제공 인프라 조건을 기준으로 해야 한다. | `local`, `private-dev`, `aws-dev`, `aws-prod`, 외부 제공 개발 환경처럼 cluster profile이 자유롭게 추가되어도 Helm/Argo CD 구조가 유지된다. | GitOps 예정 |
| `REQ.A.06.NFR-026` | 관측성 스택은 외부 SaaS 비용에 의존하지 않아야 한다. | LGTM 계열 자체 운영 스택으로 metric, log, trace, dashboard를 구성하고 저장량과 retention을 직접 관리한다. | 관측성 예정 |

## 제약 조건

- 정책/보안 제약: secret, 결제/개인정보, 운영자 권한, production debug 권한은 최소 권한과 감사 로그를 기본값으로 둔다.
- 기술 제약: DropMong은 초기에는 제한된 인원과 작은 클러스터에서 시작할 가능성이 높으므로, 과도한 platform product를 먼저 만들지 않는다.
- 일정 제약: MVP는 `local`, `private-dev`, `aws-dev` 클러스터 프로파일, Istio ingress/service mesh, Argo CD + Helm 배포, namespace/RBAC, 자체 LGTM 관측성, readiness/liveness, synthetic E2E, incident 기록을 우선한다.
- 데이터 제약: 데이터/분석 워크로드는 후속 단계에서 커질 수 있으므로, 초기에는 API workload와 섞지 않는 경계와 확장 지점만 둔다.
- 운영 제약: 피크 중 수동 DB 수정, 직접 Pod patch, 임의 secret 조회, 임의 node 접속을 일반 대응 방식으로 두지 않는다.

## 수용 기준

- 모든 서비스 Deployment에는 resource request/limit, readiness probe, liveness probe, graceful shutdown 기준이 있다.
- 외부 API는 gateway를 통과하며 route별 timeout과 rate limit을 적용할 수 있다.
- 기존 Kong ingress 기준을 Istio ingress gateway 기준으로 전환하는 요구가 문서화되어 있다.
- Istio service mesh가 내부 서비스 호출 정책의 기본 전제로 남아 있다.
- 서비스별 namespace 또는 논리적 경계와 service account가 정의되어 있다.
- 클러스터 프로파일은 `local`, `private-dev`, `aws-dev`를 초기 대상으로 포함하고, 제공 인프라 조건에 따라 새 프로파일을 추가할 수 있다.
- Argo CD는 Helm chart와 values를 동기화하며, Kustomize values는 기본 배포 경로가 아니다.
- 관측성은 Datadog이 아니라 자체 운영 LGTM 스택을 기준으로 한다.
- image pull 실패, CrashLoopBackOff, Pending, OOMKilled, readiness failure가 대시보드와 알림에서 확인된다.
- DNS error, DNS latency, CoreDNS/kube-dns saturation을 확인할 수 있다.
- 배포 후 rollback할 image tag와 GitOps 변경 이력을 확인할 수 있다.
- privileged, hostNetwork, hostPath, unsafe capability, unapproved registry가 기본 정책에서 차단된다.
- secret 원문은 문서, values, 로그, trace에 남지 않는다.
- 드롭 피크 대시보드에서 gateway traffic, service latency, error rate, queue lag, 결제/알림 실패율, pod restart를 함께 볼 수 있다.
- synthetic E2E는 Kubernetes Job으로 실행되고 실패 시 Job 상태, 로그, trace, 서비스 metric을 따라갈 수 있다.
- 온콜은 네트워크 장애 조사 시 임시 디버그 컨테이너 또는 대체 runbook을 사용할 수 있고, 권한 사용 이력이 감사 로그에 남는다.
- batch/data workload는 API workload와 별도 namespace 또는 quota로 분리된다.
- incident 기록에는 level, owner, 영향 범위, timeline, 완화 조치, 후속 action item이 남는다.

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.01](./REQ_A_01_limited_drop_commerce.md), [REQ.A.05](./REQ_A_05_auth_member.md) | 아키텍처 참조: [Kubernetes](../../../workspaces/docs/architecture/kubernetes-architecture/README.md), [Deployment](../../../workspaces/docs/architecture/deployment/README.md), [Observability](../../../workspaces/docs/architecture/observability/README.md), [Synthetic E2E](../../../workspaces/docs/architecture/synthetic-e2e/README.md) | 서비스 메시 참조: Istio 예정 | 서비스 참조: SVC.A 예정 | API 참조: API.A 예정 | GitOps 참조: Argo CD + Helm 예정 | Infra 참조: Infra 예정

## 열린 질문

- `local`, `private-dev`, `aws-dev` 다음에 어떤 제공 인프라 프로파일을 먼저 추가할 것인가?
- Istio ingress gateway 전환 시 Kong Ingress Controller는 즉시 제거할 것인가, 일정 기간 병행할 것인가?
- NodeLocal DNSCache, OPA Gatekeeper/Kyverno, external-secrets/sealed-secrets 중 어떤 항목을 MVP 기본값으로 넣을 것인가?
- LGTM 자체 운영에서 Loki/Tempo/Mimir 또는 Prometheus의 retention과 storage class는 환경별로 어떻게 다르게 둘 것인가?
- batch/data workload는 같은 클러스터에 둘 것인가, 후속 데이터 플랫폼 클러스터로 분리할 것인가?

## 확인 필요

- 예상 피크 규모: 동시 접속자, RPS, 드롭별 구매 시도 수, queue/worker 처리량
- GitOps repo의 실제 Argo CD Application tree, Helm chart, environment values 위치
- 기존 GitOps Helm chart 구성에서 `local`, `private-dev`, `aws-dev`가 각각 어떤 values 파일과 cluster secret으로 연결되는지
- Kong ingress에서 Istio ingress gateway로 전환할 때 필요한 route, certificate, DNS, load balancer 변경 범위
- registry와 imagePullSecret 운영 방식: ECR, GHCR, Docker Hub mirror, private registry 여부
- secret 관리 방식: Kubernetes Secret, sealed-secrets, external-secrets 중 선택
- 운영 대시보드의 최소 지표와 alert severity 기준
- production debug 권한 승인자와 감사 로그 보관 위치

## 새로 드러난 가장자리

- 이 자료가 답한 질문: 회사들의 공개 기술 글에서 Kubernetes 클러스터 설계가 단순 배포 대상이 아니라 DNS, image pull, 네트워크 진단, 보안 정책, 관측성, 데이터 워크로드, 인시던트 절차까지 포함하는 운영 제품이라는 점을 확인했다. 또한 DropMong은 `local`, `private-dev`, `aws-dev`를 시작점으로 Argo CD + Helm, Istio, 자체 LGTM을 깊게 운영하는 학습/포트폴리오 방향을 갖는다.
- 아직 남은 질문: DropMong MVP에서 admission policy, secret operator, DNS cache, batch/data workload 분리를 어디까지 기본값으로 넣을지 결정되지 않았다.
- 검증되지 않은 전제: DropMong은 초기에는 단일 region, active cluster profile별 GitOps 배포를 사용한다고 가정했다.
- 약한 연결: Spark/ML 서빙 요구는 현재 DropMong 핵심 구매 경로보다 후순위일 수 있어 데이터/AI 기능이 실제 범위에 들어올 때 우선순위를 조정해야 한다.
- 다음에 확인할 것: GitOps tree와 Helm values 위치, Istio ingress 전환 범위, 자체 LGTM retention/storage, secret 관리, production debug 승인 절차

## 레퍼런스

- DropMong: [Kubernetes architecture](../../../workspaces/docs/architecture/kubernetes-architecture/README.md)
- DropMong: [Deployment architecture](../../../workspaces/docs/architecture/deployment/README.md)
- DropMong: [Observability architecture](../../../workspaces/docs/architecture/observability/README.md)
- DropMong: [Synthetic E2E architecture](../../../workspaces/docs/architecture/synthetic-e2e/README.md)
- Mercari: [From DNS Failures to Resilience](https://engineering.mercari.com/en/blog/entry/20250515-from-dns-failures-to-resilience-how-nodelocal-dnscache-saved-the-day/)
- Mercari: [When Caching Hides the Truth](https://engineering.mercari.com/en/blog/entry/20250523-when-caching-hides-the-truth-a-vpc-service-controls-artifact-registry-tale/)
- Mercari: [Capturing Network Packets in Kubernetes](https://engineering.mercari.com/en/blog/entry/20251218-capturing-network-packets-in-kubernetes/)
- Mercari: [Enhance Kubernetes Security with OPA Gatekeeper](https://engineering.mercari.com/en/blog/entry/20201222-enhance-kubernetes-security-with-opa-gatekeeper/)
- Mercari: [The Cost of Speed](https://engineering.mercari.com/en/blog/entry/20251215-the-cost-of-speed-a-battle-against-cost-debt-and-diverging-systems/)
- Bucketplace: [오늘의집 Spark on Kubernetes 도입 및 개선 여정](https://www.bucketplace.com/post/2025-05-23-%EC%98%A4%EB%8A%98%EC%9D%98%EC%A7%91-spark-on-kubernetes-%EB%8F%84%EC%9E%85-%EB%B0%8F-%EA%B0%9C%EC%84%A0-%EC%97%AC%EC%A0%95/)
- Bucketplace: [개인화 추천 시스템 #3. 모델 서빙](https://www.bucketplace.com/post/2025-03-14-%EA%B0%9C%EC%9D%B8%ED%99%94-%EC%B6%94%EC%B2%9C-%EC%8B%9C%EC%8A%A4%ED%85%9C-3-%EB%AA%A8%EB%8D%B8-%EC%84%9C%EB%B9%99/)
- Viva Republica: [Spark Connect on Kubernetes #1](https://toss.tech/article/spark-connect-on-kubernetes-1)
- Oliveyoung: [올리브영은 인시던트를 어떻게 관리하고 있는가?](https://oliveyoung.tech/2024-01-23/incident/)
- Oliveyoung: [Oliveyoung at DASH 2024](https://oliveyoung.tech/2024-08-05/dash-2024-slide/)
- Oliveyoung: [Host Level Chaos Engineering](https://oliveyoung.tech/2026-03-30/chaos-host-level/)
