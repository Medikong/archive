# Zalando 기술 자료 모음
> e커머스 파이널 프로젝트 참고자료 — 인프라/CI·CD/DevOps 관점
> 작성일: 2026-07-02

---

## 1. 원문 요약: Fashion Search and Discovery 마이크로서비스

출처: [engineering.zalando.com — Using Microservices to Power Fashion Search and Discovery](https://engineering.zalando.com/posts/2017/02/using-microservices-to-power-fashion-search-and-discovery.html) (2017.02)

### 글의 목적
패션 검색·발견 같은 상태 저장(stateful) 솔루션에 마이크로서비스 패턴을 적용한 원칙을 설명. 시스템 분해는 소비자 사용 사례(유스케이스)로 기능 경계를 먼저 설정하는 데서 출발.

### 핵심 소비자 사용 사례
- 제품 검색: 카테고리 트리를 진입점으로 카탈로그 접근. 프로필·선호도·검색기록에 따라 결과가 달라지는 암묵적 문맥 검색
- 결과 정제: 필터(정적, 오프라인 구축)와 패싯(동적, 상호작용마다 재구축)으로 결과 좁히기
- 관련성 정렬: 정적 요소(브랜드·카테고리) + 동적 요소(재고·판매자 순위·배송시간) 기반 순위화
- 의도 분석: 비정형 의도(키워드·시각검색)를 반정형 쿼리로 변환
- 개인화: 소비자 행동 분석으로 프로필 구축, 맞춤 제품 제공

### 비기능 요구사항
- 상호 운용성: 공통 데이터 모델 기반 인터페이스
- 진화: 기존 데이터 계약 유지하면서 새 메타데이터 수용
- 일관성: 분산시스템 특성상 100% 일관성은 불가, 가격·재고 속성의 최대 전파 지연 30분으로 정의 (도메인 중심 설계 준수)
- 지연 시간: 사람↔기기 상호작용 1초 이내, 마이크로서비스 단위 100ms 이내
- 확장성: 스토리지 계층 수평 확장, 비즈니스 성장의 제약이 되면 안 됨
- 가용성: 데이터 가용성·내구성, 이벤트 기반 카탈로그의 궁극적 요구사항

### 엔드투엔드 아키텍처 (4계층)
| 계층 | 역할 | 상태 |
|------|------|------|
| 스토리지 계층 | 핵심 데이터 저장, TF-IDF 정보검색 시스템 기반, AWS로 내구성·가용성 강화, 고가용성+자동 복구 | Stateful |
| 콘텐츠 수집 계층 | ETL 수행하는 이벤트 기반 통합 파이프라인, 동기·비동기 혼용 | 혼합 |
| 검색 계층 | 소비자 여정에 집중하는 순수 마이크로서비스("검색 비즈니스 로직") | Stateless |
| 소비자 대면 애플리케이션 | BFF/리버스 프록시, API 진화를 서비스 진화와 분리 | - |

### 인터페이스 (REST만으로는 부족 → 4종 혼용)
- REST: HTTP+JSON 동기 통신 표준
- MQi: 비동기 pub/sub (AWS SQS, Kinesis, Nakadi)
- KVi: 키-값(해시맵/블롭) 접근
- QLi: 벤더 종속 검색 인터페이스 (Elasticsearch/SolrCloud), TF-IDF "핫 스왑" 가능
- CSi: 클릭스트림 이벤트 게시
- 단, API-first 설계 원칙은 유지

### 주요 마이크로서비스
- 콘텐츠 크롤러: 여러 소스 집계·정규화(CDM), Last-Value-Caching으로 로컬 데이터 가용성 보장, replay로 데이터 손실 복구
- 콘텐츠 분석: 패션 특징 추출 → 검색 가능한 스니펫 생성, 토크나이저·스테밍·텍스트 분석 파이프라인
- 카탈로그: TF-IDF 인덱스 모음, 분당 수백만 요청 탄력성 목표
- 패션 메타데이터: 계층 범주·필터·패싯 제공
- 구조화된 검색 / 쿼리 플래너: 의도를 패턴 매칭/쿼리로 변환 (TF-IDF 기술과 분리 설계)
- 관련성 정렬: 전처리(쿼리 재구성)·후처리(결과 재정렬)로 개인화
- 제품 게이트웨이: 검색된 제품에 파트너 재고 등 메타데이터 보강
- 소비자 분석: 검색 행동을 구매 의도와 연결("소비자 게놈")

### 공통 데이터 모델(CDM)
- 콘텐츠를 일반적 어휘·구조로 정의해 애플리케이션↔플랫폼 상호 운용성 보장
- 도메인 객체를 평면 구조(인접 리스트)로 분할: 각 조각은 타입·참조(전역 고유 식별자)·속성을 가짐
- 여러 소스(브랜드·소매업체 피드)의 JSON 키 충돌 문제 해결

### 결론
구성 요소가 많아 보이지만 기술적 문제를 마이크로서비스 단위로 분리하기 위한 의도적 분해. REST 일괄 적용·공통 데이터 모델이 일부 마이크로서비스 방식과 상충함을 발견. 이 아키텍처로 2016년 블랙프라이데이의 대규모 검색 수요를 성공적으로 처리.

---

## 2. 참고 기술 블로그 · 자료 모음

### 최우선 추천

**How Zalando manages 140+ Kubernetes Clusters**
출처: [srcco.de](https://srcco.de/posts/how-zalando-manages-140-kubernetes-clusters.html) (Henning Jacobs, Head of Developer Productivity)

프로젝트의 인프라/CI·CD/운영 영역에 가장 직접적으로 쓸모 있는 글.

- 목표: No manual operations(모든 클러스터 업데이트·운영 완전 자동화), No pet clusters(클러스터는 다 똑같이 생겨야 함), 신뢰성, 오토스케일링
- 변경 프로세스: "dev" 브랜치에 PR을 여는 것으로 시작 → 자동 e2e 테스트(공식 K8s conformance test + Zalando 자체 테스트, 35~59분 소요) 통과 + 사람 승인("+1") 후 머지 — Branch Protection Ruleset과 동일 패턴
- 각 변경은 채널(git 브랜치)을 거쳐 stable 채널의 모든 프로덕션 클러스터까지 단계적으로 이동
- "production-ready" 정의: 신뢰된 빌드 시스템(내부 CD 플랫폼)으로 빌드 + 최소 2명 엔지니어 코드 리뷰 승인
- 클러스터는 항상 쌍(production + non-prod)으로, 도메인/product community별 완전히 격리된 AWS 계정에 프로비저닝
- "you build it, you run it" — 200개 이상 개발팀이 24/7 온콜 포함 앱을 직접 책임
- 관찰성: ZMON(메트릭·알림·OpsGenie 페이징·Grafana 대시보드), OpenTracing(LightStep) 분산추적, 중앙 로깅(Scalyr), kube-resource-report, kube-web-view
- 표준 K8s API 인증에 Zalando OAuth 토큰, Ingress는 ALB Ingress Controller + Skipper 사용

AWS re:Invent 발표 영상도 존재(Henning Jacobs). 블로그 내용과 거의 동일.

### 인프라 · 클러스터 관리

**Zalando Case Study — Kubernetes 공식**
출처: [kubernetes.io/case-studies/zalando](https://kubernetes.io/case-studies/zalando/)

- 2015년 자율 자기조직 팀 체제로 전환하며 AWS CloudFormation 운영 오버헤드 문제로 클러스터 관리 도입 결정
- 2016년 Hack Week에서 K8s 배포 시작 → 60명 기술인프라팀 온보딩 → 팀별 순차 도입
- 도입 전 K8s 교육은 대부분 CI/CD 셋업 교육 — 개발자 인터페이스가 곧 CI/CD 시스템이라는 관점

**GitHub: zalando-incubator/kubernetes-on-aws**
- CloudFormation + Ubuntu로 AWS에 K8s 프로비저닝하는 설정 템플릿
- Cluster Registry(원하는 상태 저장) + Cluster Lifecycle Manager(프로비저닝·매니페스트 적용) + Cluster Lifecycle Controller(내부 롤링 업데이트 처리) 구조
- Full Ingress 지원: ALB/NLB+TLS via kube-ingress-aws-controller, HTTP 라우팅 via Skipper
- stackset-controller + Skipper로 managed stack, blue-green 배포 지원

**GitHub: zalando-incubator/cluster-lifecycle-manager (CLM)**
- Cluster Registry + git 설정 저장소를 읽어 클러스터를 최신 상태로 유지
- Reentrant 설계 — 언제 죽어도 중단 지점부터 재개
- 2017년 1월부터 내부 개발, 200개 이상 클러스터를 K8s v1.4~v1.24까지 연속 업데이트
- 참고 발표: 2018 KubeCon EU "Continuously Deliver your Kubernetes Infrastructure" (Mikkel Larsen)

### 데이터베이스 (재고/주문 DB 설계 시)

**Postgres Operator**
출처: [github.com/zalando/postgres-operator](https://github.com/zalando/postgres-operator), [opensource.zalando.com/postgres-operator](https://opensource.zalando.com/postgres-operator/docs/administrator.html)

- Postgres 매니페스트(CRD)만으로 설정, K8s API 직접 접근 없이 CI/CD 파이프라인에 통합 — IaC 지향
- Patroni 기반 자동 failover, 무중단 볼륨 리사이즈(AWS EBS, PVC), 빠른 in-place 메이저 버전 업그레이드
- 자식 리소스에 owner reference 설정 → GitOps 도구 모니터링 개선 + cascade 삭제 가능 (단, PVC는 StatefulSet의 PV Reclaim Policy로 별도 처리, 네임스페이스 간 시크릿은 owner reference 불가)
- 5년 이상 프로덕션 운영 검증됨

### 오픈소스 생태계

- Zalando는 GitHub에 200개 이상의 오픈소스 프로젝트 보유, 매달 약 2개 신규 프로젝트 제안
- 대표 프로젝트: Skipper(HTTP 라우팅 프록시, 안정적 K8s 클러스터·스케일링 인그레스 컨트롤러), Postgres Operator
- 블로그 자체 회고: Elasticsearch 대규모 설치 운영 장애 사후분석(root cause analysis) 글 존재 — 장애 보고서 자료로 활용 가능

### GitOps 일반 참고 (Zalando 외부, 개념 보충용)

**Multi-Cluster GitOps Deployment Using Argo CD**
출처: [Medium — Lekhana Vemuri](https://medium.com/@lekhanavemuri/multi-cluster-gitops-deployment-using-argo-cd-on-kubernetes-f5bd60922bdf) (2026.06)
- CNCF 2025 설문 기준 약 60%의 K8s 클러스터가 ArgoCD 사용
- 단일 중앙 ArgoCD 인스턴스가 kubeconfig로 여러 클러스터를 원격 연결해 관리하는 멀티클러스터 아키텍처 사례

---

## 3. 영역별 추천 요약

| 담당 영역 | 추천 자료 | 핵심 인용 포인트 |
|---|---|---|
| 인프라 설계 전반 | Kubernetes.io Zalando Case Study | 클러스터 관리 도입 배경, 팀 온보딩 프로세스 |
| CI/CD | srcco.de "140+ Kubernetes Clusters" | dev→stable 채널 단계적 배포, e2e 테스트 게이트, "+1" 승인 |
| 운영·관찰성 | 위 글의 ZMON/OpenTracing/Scalyr 섹션 | 메트릭·분산추적·로깅 스택 구성 |
| DB 설계 심화 | Postgres Operator (GitHub) | CRD 기반 IaC, Patroni HA, 무중단 리사이즈 |
| 도메인 분해 근거 | Fashion Search 원문 (본 문서 1절) | Bounded Context, SLO 수치(30분/100ms), 4계층 구조 |

시간이 부족한 경우: srcco.de 글 1개 + Fashion Search 원문 1개만 정독해도 CI/CD·운영·도메인 설계 근거를 대부분 커버할 수 있음.

블로그 태그 페이지에서 최신 글도 확인 가능: https://engineering.zalando.com/tags/kubernetes.html
