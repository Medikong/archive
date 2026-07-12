# DropMong 멘토링 피드백 정리

작성일: 2026-07-07

## 목적

이 문서는 2026-07-07 멘토링에서 받은 피드백을 DropMong 설계 검토 과제로 남긴다.

핵심 피드백은 다음 한 문장으로 정리할 수 있다.

```text
겉으로 보이는 아키텍처 구성을 넘어서,
왜 그렇게 나누었는지, 실패했을 때 어떻게 알리고 되돌릴지,
트래픽과 비용 조건 안에서 어떤 선택을 포기하거나 유지할지를 설명해야 한다.
```

이번 피드백은 구현 변경 지시가 아니라 후속 설계 검토와 조사 과제다. 각 항목은 이후 별도 문서, 실험, 검증 기준으로 나눠 정리한다.

## 핵심 방향

| 주제 | 받은 피드백 | 해석 |
| --- | --- | --- |
| Kubernetes namespace | namespace가 너무 많이 나뉜 것으로 보인다 | 왜 나누었는지, 다른 선택지는 없는지 근거가 필요하다 |
| NetworkPolicy / Ingress | 서비스마다 정책이 나뉘는 것이 적절한지 의문 | 보안 경계, 운영 복잡도, 실제 회사 사례 기준으로 다시 검토한다 |
| 관측성 | 구축했다는 사실보다 제대로 수집되고 동작하는지가 중요하다 | 성공/실패 기준과 알림, rollback 조건이 필요하다 |
| 모니터링 비용 | 관측성 구성 자체가 CPU/메모리를 많이 쓸 수 있다 | 경량화 옵션과 위험한 기능 비활성화 후보를 검토한다 |
| 감사 로그 | Kubernetes 감사 로그 저장과 보관 주기가 필요하다 | 보관 기간, 2차 백업, 저장소 장애 시 복구 계획을 정해야 한다 |
| 동시성 제어 | DB 앞단에서 connection/thread/concurrency 제한이 필요하다 | DB connection pool만 보지 말고 서비스 앞단의 backpressure를 검토한다 |
| 일정/R&R | 시간이 부족하면 무엇을 포기할지 정해야 한다 | 테스트, QA, 실험, 발표 자료의 우선순위를 나눈다 |
| 트래픽 시나리오 | 동시접속자가 유동적일 때의 시나리오가 필요하다 | 타임딜에서 발생할 이슈를 먼저 리스트업하고 벤치마킹한다 |
| 인프라/비용 | 노드, DB 이중화, control plane 구성을 비용과 연결해야 한다 | 기술 선택을 예상 수익, 운영 비용, 사용자 범위와 함께 본다 |
| 프로젝트 취지 | 기술적인 고민만으로 끝나면 안 된다 | 프로젝트 목표에 맞춘 기획, 설계, 검증 계획으로 연결한다 |

## 1. Namespace와 네트워크 경계 재검토

멘토링 메모:

- Kubernetes namespace가 너무 많이 나뉘어져 있다.
- namespace를 나눈 이유와 근거를 찾아야 한다.
- Ingress와 NetworkPolicy를 함께 봐야 한다.
- 동일한 ingress가 namespace마다 생길 수도 있다.
- NetworkPolicy가 6개로 나뉘어질 때 정말 6개 분리가 필요한지 검토해야 한다.
- 회사 프로젝트에서는 개별 서비스마다 namespace나 정책을 나누지 않는 경우도 많다.
- 다른 방법을 고민해야 한다.

정리:

현재 구조가 서비스별 namespace나 세분화된 NetworkPolicy를 전제로 한다면, 단순히 "분리하면 안전하다"는 설명만으로는 부족하다. namespace는 보안 격리, 권한 위임, 배포 단위, 리소스 quota, 관측성 분류에는 도움이 되지만, 서비스 수만큼 늘어나면 Ingress, DNS, NetworkPolicy, ServiceMonitor, Secret, 배포 자동화가 함께 복잡해진다.

검토할 대안:

| 대안 | 설명 | 장점 | 위험 |
| --- | --- | --- | --- |
| 단일 application namespace | dev 환경의 앱 서비스를 하나의 namespace에 둔다 | 운영 단순성, 정책 관리 쉬움 | 서비스별 강한 격리는 약함 |
| 도메인별 namespace | auth, commerce, observability 등 책임 영역으로 나눈다 | 서비스별보다 덜 복잡하고 의미 있는 경계 | 도메인 기준 합의 필요 |
| 환경별 namespace | dev, staging, prod처럼 환경 단위로 나눈다 | 작은 팀에서 관리하기 쉽다 | prod 내 세부 격리는 별도 정책 필요 |
| 민감 서비스만 분리 | auth, DB 접근 서비스 등만 별도 namespace로 둔다 | 보안과 단순성 균형 | 분리 기준이 흔들릴 수 있음 |
| 서비스별 namespace | 각 서비스마다 namespace를 둔다 | 격리와 quota는 명확 | 작은 프로젝트에서는 운영 비용이 큼 |

후속 질문:

- 현재 namespace 분리 기준은 서비스 책임인가, 보안 등급인가, 배포 단위인가?
- 같은 Ingress/Gateway가 namespace마다 중복되는 구조인가?
- NetworkPolicy는 실제로 차단해야 할 통신을 정의하고 있는가, 아니면 형식적으로 생성된 것인가?
- 작은 팀과 제한된 기간에서 서비스별 namespace가 얻는 이익이 운영 비용보다 큰가?

## 2. 관측성의 성공과 실패 기준

멘토링 메모:

- "관측성을 구축해봤다" 정도로 보일 수 있다.
- 관측성 옵션을 하나하나 이해하기 쉽지 않다.
- 제대로 동작했는지, 수집이 되었는지 기준이 필요하다.
- 안 됐을 때 알림은 무엇인지, 안 됐을 때 rollback은 무엇인지 정해야 한다.
- 배포의 성공과 실패 기준이 필요하다.
- 결과값을 어디까지 관제받을 것인지 정해야 한다.

정리:

관측성은 Prometheus, Loki, Tempo, Grafana를 설치했다는 사실보다 "운영 질문에 답할 수 있는가"가 중요하다. 배포 성공/실패 기준, 알림 조건, rollback 조건이 없으면 대시보드는 있어도 운영 체계는 없는 상태가 된다.

정의해야 할 기준:

| 영역 | 필요한 기준 |
| --- | --- |
| 수집 성공 | metric, log, trace가 어떤 label과 주기로 들어오는가 |
| 배포 성공 | error rate, latency, readiness, business metric이 어느 범위 안에 있어야 하는가 |
| 배포 실패 | 몇 분 동안 어떤 지표가 기준을 넘으면 실패로 볼 것인가 |
| 알림 | 누구에게, 어떤 채널로, 몇 단계 심각도로 보낼 것인가 |
| rollback | 수동 rollback인지, Argo Rollouts 같은 자동 rollback인지 |
| 관제 범위 | 시스템 지표만 볼 것인지, 비즈니스 결과까지 볼 것인지 |

후속 질문:

- DropMong의 첫 번째 SLO는 무엇인가?
- 쿠폰 발급, 품절, 중복, Redis fallback은 각각 어떤 지표로 관측할 것인가?
- 배포 후 몇 분 동안 어떤 지표를 보고 성공으로 판단할 것인가?
- 장애 알림을 받았을 때 사람이 할 행동이 문서화되어 있는가?

## 3. 모니터링 시스템의 리소스 비용과 경량화

멘토링 메모:

- 모니터링 서비스들이 CPU와 메모리를 많이 잡아먹는다.
- 모니터링 시스템 때문에 장애가 발생하는 상황도 생긴다.
- 모니터링 서비스 경량화가 필요하다.
- 위험성이 있는 옵션은 무엇을 비활성화해야 하는지 검토해야 한다.

정리:

관측성 스택은 서비스 안정성을 높이기 위해 넣지만, 작은 클러스터에서는 Prometheus, Loki, Tempo, Grafana, Collector가 리소스를 크게 차지할 수 있다. 따라서 "무엇을 볼 것인가"뿐 아니라 "무엇을 수집하지 않을 것인가"도 설계해야 한다.

검토할 경량화 후보:

| 영역 | 검토 후보 |
| --- | --- |
| metrics | scrape interval 증가, high-cardinality label 제거, 불필요한 exporter 비활성화 |
| logs | 로그 샘플링, debug 로그 기본 비활성화, 보관 기간 단축 |
| traces | trace sampling 적용, 모든 요청 full tracing 금지 |
| dashboard | 과도한 query panel 제거, expensive query 점검 |
| storage | retention 단축, 압축, 외부 저장소 여부 검토 |
| collector | processor/exporter 최소화, batch/queue 설정 검토 |

후속 질문:

- dev 환경에서 관측성 스택에 줄 수 있는 CPU/메모리 상한은 얼마인가?
- 어떤 로그와 trace는 수집하지 않아도 되는가?
- 장애 분석에 꼭 필요한 최소 signal은 무엇인가?
- 관측성 장애가 애플리케이션 장애로 번지지 않게 할 격리 방법은 무엇인가?

## 4. 로그 샘플링과 Kubernetes 감사 로그

멘토링 메모:

- 검증 항목으로 로그 샘플링이 필요하다.
- Kubernetes 자체의 감사 로그도 있다.
- 감사 로그 데이터를 어떻게 저장할지 정해야 한다.
- 보관 주기를 어떻게 둘지 정해야 한다.
- 추상적으로 2달이라고 잡는 것만으로는 부족하다.
- 2차 백업을 할 것인지 검토해야 한다.
- 저장 공간은 서버나 NAS일 수 있다.
- 저장 공간에 물리적 문제가 생겼을 때 어떻게 할지 정해야 한다.

정리:

로그 정책은 애플리케이션 로그와 Kubernetes audit log를 분리해서 봐야 한다. 애플리케이션 로그는 장애 분석과 사용자 요청 추적에 쓰이고, audit log는 누가 cluster resource를 변경했는지 확인하는 보안/운영 증거다.

정해야 할 것:

| 항목 | 결정 필요 |
| --- | --- |
| 애플리케이션 로그 | sampling 기준, 필수 필드, 보관 기간 |
| 감사 로그 | 수집 범위, 저장 위치, 검색 방식 |
| 보관 주기 | 2달 기준의 근거, 비용, 용량 산정 |
| 2차 백업 | NAS, 외부 스토리지, 별도 서버 중 선택 |
| 장애 대응 | 저장소 장애 시 로그 유실 허용 범위와 복구 방식 |

후속 질문:

- 2달 보관은 법적/보안 요구인가, 운영 편의 기준인가?
- audit log까지 Loki에 넣을 것인가, 별도 파일/스토리지로 보관할 것인가?
- 로그 저장소가 가득 찼을 때 애플리케이션에 영향을 주지 않도록 되어 있는가?
- 2차 백업이 필요한 로그와 필요 없는 로그를 나눌 수 있는가?

## 5. DB 앞단의 동시성 제어

멘토링 메모:

- connection pool 할당과 해제를 봐야 한다.
- DB 전에 앞단 thread pool이나 concurrency limit 처리가 필요할 수 있다.
- 요청이 DB까지 모두 들어가기 전에 제한해야 한다.

정리:

DB connection pool만 조절하면 이미 DB 앞까지 요청이 밀려온 뒤에야 제한된다. 피크 트래픽에서는 서비스 내부 worker/thread pool, HTTP handler concurrency, Redis gate, queue, circuit breaker, backpressure를 함께 봐야 한다.

검토할 계층:

| 계층 | 검토할 제어 |
| --- | --- |
| Gateway / Ingress | rate limit, timeout, request body limit |
| Service handler | in-flight request limit, timeout, cancellation |
| Redis gate | sold-out/duplicate 조기 차단 |
| DB access | connection pool max, transaction timeout |
| 비동기 처리 | queue length, consumer concurrency, retry backoff |

후속 질문:

- coupon issue 경로에서 동시에 DB transaction까지 들어갈 수 있는 최대 요청 수는 얼마인가?
- DB pool 고갈 시 사용자에게 어떤 응답을 줄 것인가?
- backpressure를 어디에서 시작하는 것이 가장 효과적인가?
- circuit breaker가 필요한 외부 의존성이 있는가?

## 6. 일정, R&R, 포기할 범위

멘토링 메모:

- 일정 계획과 R&R을 나눠봐야 한다.
- 일정상 안 되면 여러 가지를 포기해야 한다.
- 테스트나 QA 시간을 줄여야 할 수도 있다.

정리:

프로젝트는 기술적으로 하고 싶은 것을 모두 넣는 것이 아니라, 기간 안에서 보여줄 핵심 가치를 정해야 한다. 특히 테스트와 QA 시간을 줄이는 것은 단기적으로 시간을 벌 수 있지만, 멘토링에서 받은 피드백의 방향과는 반대일 수 있다. 설계의 신뢰도를 보여주려면 오히려 작은 범위를 정하고 그 범위를 검증하는 편이 낫다.

우선순위 기준:

| 우선순위 | 남길 것 | 줄일 수 있는 것 |
| --- | --- | --- |
| P0 | 사용자 여정, 핵심 서비스 경계, Redis gate 검증, 최소 관측성 기준 | 부가 기능 |
| P1 | 부하 시나리오, 알림/rollback 기준, namespace 재검토 | 과도한 대시보드 |
| P2 | 장기 확장 설계, 비용 시뮬레이션, HA 구성 비교 | 발표용 장식 자료 |

후속 질문:

- 이번 프로젝트의 최종 평가 기준은 구현 범위인가, 설계 근거인가, 검증 결과인가?
- 역할을 나눈다면 누가 기능, 누가 배포, 누가 관측성, 누가 QA를 맡는가?
- 시간이 부족할 때 절대 포기하지 않을 검증은 무엇인가?

## 7. 동시접속자 시나리오와 타임딜 이슈 목록

멘토링 메모:

- 동시접속자가 유동적일 때 가져갈 수 있는 시나리오를 토대로 아키텍처를 검토할 수 있다.
- 한정된 시간이 있으니 벤치마킹이 필요하다.
- 타임딜에서 이슈가 될 만한 내역을 1차적으로 리스트업하는 것이 중요하다.

정리:

아키텍처 검토는 "사용자가 많다"가 아니라 구체적인 시나리오에서 시작해야 한다. 예를 들어 오픈 1분 전 조회 증가, 오픈 직후 발급 폭주, 품절 이후 재시도 폭주, 결제 실패, 알림 지연처럼 나누어야 한다.

1차 이슈 목록:

| 시나리오 | 주요 위험 | 검증 지표 |
| --- | --- | --- |
| 오픈 전 조회 증가 | catalog/cache 부하 | read latency, cache hit ratio |
| 오픈 직후 발급 폭주 | DB row lock, connection pool 고갈 | p95/p99, DB finalize count |
| 중복 클릭/재시도 | 중복 발급, idempotency 오류 | duplicate count, replay consistency |
| 품절 이후 폭주 | 불필요한 DB 진입 | sold_out latency, Redis gate hit |
| Redis 장애 | fallback으로 DB 과부하 | fallback count, DB saturation |
| 결제 실패/지연 | 예약 해제 실패 | release count, expired order |
| 알림 지연 | 핵심 흐름과 알림 결합 | notification lag, DLQ |

후속 질문:

- 발표 전까지 어떤 시나리오 하나를 수치로 검증할 것인가?
- 동시접속자 수를 몇 단계로 나누어 볼 것인가?
- 성공 기준은 평균 latency가 아니라 어떤 percentile과 error budget으로 볼 것인가?

## 8. 인프라 HA, 비용, 사업성 관점

멘토링 메모:

- Kubernetes control plane을 한 대로 할 것인지 검토해야 한다.
- DB를 이중화할 것인지 검토해야 한다.
- 노드를 늘릴 것인지, circuit breaker, backpressure를 둘 것인지 정답은 없다.
- 서비스 목표, 사용자 범위, 트래픽 범위, 운영 비용을 함께 검토해야 한다.
- 예상 수입과 한 달 인프라 비용을 봐야 한다.
- 초기부터 인프라를 크게 구성하고 사용자가 늘어날 것을 예상할 수도 있다.
- 기술적인 고민만 할 게 아니라 장기 확장성과 돈을 버는 구조를 함께 봐야 한다.

정리:

인프라 구성은 기술적으로 가장 안전한 선택이 항상 정답은 아니다. control plane HA, DB 이중화, 노드 증설은 가용성을 높이지만 비용과 운영 복잡도도 늘린다. DropMong이 실제 서비스라고 가정한다면 예상 트래픽, 매출, 장애 허용 범위, 운영 인력, 월 인프라 비용을 함께 봐야 한다.

검토할 결정:

| 결정 | 낮은 비용 선택 | 높은 안정성 선택 | 판단 기준 |
| --- | --- | --- | --- |
| control plane | single control plane | multi control plane | 운영 환경인지, 학습/검증 환경인지 |
| DB | single primary | replica/managed HA | 데이터 손실 허용 범위, 복구 시간 |
| node | 작은 노드 수 | 수평 확장 여유 | 피크 트래픽 예측, 비용 |
| 장애 대응 | 수동 대응 | 자동 rollback/auto scaling | 운영 인력, 장애 허용 시간 |
| 관측성 | 최소 수집 | full metric/log/trace | 분석 필요성과 리소스 비용 |

후속 질문:

- 월 인프라 비용 상한은 얼마로 잡을 것인가?
- 초기 사용자 수와 피크 동시접속자를 어느 정도로 가정할 것인가?
- 장애 한 번이 매출과 사용자 신뢰에 미치는 비용은 어느 정도인가?
- 학습 프로젝트와 실제 서비스 가정 중 어느 쪽에 더 무게를 둘 것인가?

## 9. 프로젝트 취지와 기획 설계

멘토링 메모:

- 프로젝트 취지에 맞춰서 계획한 것에 대한 기획과 설계가 필요하다.
- 기술적인 고민만 하면 안 된다.

정리:

DropMong은 기술 스택 전시가 아니라 "한정 수량 커머스에서 발생하는 문제를 어떤 순서로 정의하고 검증하는가"를 보여주는 프로젝트가 되어야 한다. 따라서 설계 문서는 서비스 구조뿐 아니라 사용자 문제, 트래픽 가정, 비용 가정, 검증 기준, 포기한 범위를 함께 담아야 한다.

문서화할 질문:

- DropMong은 어떤 사용자의 어떤 문제를 해결하는가?
- MVP에서 반드시 성공해야 하는 사용자 흐름은 무엇인가?
- 어떤 기술 선택이 이 문제 해결에 직접 연결되는가?
- 어떤 기능은 지금 만들지 않는가?
- 검증 결과가 나오면 다음 의사결정은 무엇인가?

## 후속 산출물 후보

| 우선순위 | 문서 후보 | 목적 |
| --- | --- | --- |
| P0 | `namespace-network-policy-review.md` | namespace, ingress, NetworkPolicy 분리 근거와 대안 정리 |
| P0 | `observability-success-failure-criteria.md` | 배포 성공/실패, 알림, rollback 기준 정리 |
| P0 | `timedeal-risk-scenarios.md` | 타임딜/드롭 커머스 위험 시나리오와 벤치마킹 기준 정리 |
| P1 | `observability-resource-lightweight-plan.md` | 관측성 스택 리소스 비용과 비활성화 후보 정리 |
| P1 | `audit-log-retention-backup-plan.md` | Kubernetes audit log 보관 주기와 2차 백업 계획 |
| P1 | `db-concurrency-backpressure-plan.md` | connection pool, concurrency limit, backpressure 설계 |
| P2 | `infrastructure-cost-ha-decision.md` | control plane, DB HA, node 증설, 월 비용 의사결정 |
| P2 | `project-schedule-rr-scope.md` | 일정, R&R, 포기할 범위, QA 시간 계획 |

## 바로 할 일

1. 현재 namespace, ingress, NetworkPolicy 구성을 실제 파일 기준으로 inventory한다.
2. 서비스별 namespace가 필요한 근거와 대안을 표로 비교한다.
3. DropMong의 P0 운영 질문을 5개 이하로 줄인다.
4. 쿠폰 발급 경로의 성공/실패 지표와 알림 조건을 먼저 정의한다.
5. 타임딜 위험 시나리오를 기준으로 첫 benchmark scope를 정한다.
6. 관측성 스택이 사용하는 CPU/메모리를 측정하고 경량화 후보를 고른다.
7. 일정과 R&R을 나누고, 시간이 부족할 때 포기할 범위를 정한다.

