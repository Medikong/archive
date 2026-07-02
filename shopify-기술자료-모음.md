# Shopify 기술 자료 모음
> e커머스 파이널 프로젝트 참고자료 — 인프라/CI·CD/DevOps 관점
> 작성일: 2026-07-02

---

## 1. 핵심 정체성: "겉은 Rails 앱, 속은 분산 시스템"

Shopify는 Zalando와 정반대 노선. MSA로 가지 않고 모놀리스를 유지하면서, 인프라 레이어에서 격리·확장 문제를 해결함.

개발자 입장에서는 평범한 Rails 앱처럼 보이지만(DB 마이그레이션 하나가 100개 이상의 DB 샤드에 무중단으로 비동기 적용됨), 내부적으로는 정교한 격리 인프라가 이를 지탱함. CI·테스트·배포 등 인프라 전 영역이 동일한 철학으로 설계되어 있음.

---

## 2. Pods 아키텍처 — 장애 격리의 핵심

출처: [A Pods Architecture To Allow Shopify To Scale](https://shopify.engineering/a-pods-architecture-to-allow-shopify-to-scale) (Shopify Engineering)

### 배경 — 샤딩만으로는 부족했던 이유
2015년 단일 DB 서버 확장 한계에 도달 → DB를 샤딩해서 수평 확장. 하지만 `Sharding.with_each_shard do ... end` 같은 코드 패턴 때문에 샤드 하나가 죽으면 그 작업 자체가 전체 플랫폼에서 불가능해지는 문제가 발생. 샤드 수가 늘수록 이 리스크는 커짐.

### 해결책 — Pod
Pod는 완전히 격리된 데이터스토어 세트(MySQL, Redis, Memcached)를 가진 상점(shop) 묶음. 어느 리전에도 생성 가능.

- Job worker·앱서버·로드밸런서 같은 공유 리소스는 있지만, 한 번에 하나의 Pod하고만 통신 가능 — Pod 간 크로스 커뮤니케이션 금지
- 애플리케이션 서버는 "Sorting Hat"이 붙인 헤더로 어느 Pod의 데이터스토어에 연결할지 판단
- Pod마다 데이터센터 쌍(활성+복구용) 할당 → "Pod Mover"로 요청·작업 유실 없이 1분 안에 Pod를 복구 데이터센터로 이동 가능
- 현재 100개 이상의 Pod 운영 중, 도입 후 전체 장애가 아닌 단일 Pod/리전 장애로 국한됨

핵심 통찰: "Kubernetes Pod와 이름은 같지만 다른 개념". Shopify의 Pod는 cell-based 아키텍처의 cell과 거의 동일한 발상 — 하나가 죽어도 전체가 안 죽게 blast radius를 미리 제한하는 것.

### Redismageddon — SPOF 실제 사례
샤딩 이후에도 모든 샤드가 단일 Redis 인스턴스를 공유하고 있었음. 이 Redis 하나가 죽자 Shopify 전체가 다운되는 사건이 발생("Redismageddon"). 교훈은 "전체에서 공유되는 리소스는 무엇이든 피하라"였고, 이후 Redis도 Pod 모델에 맞춰 Pod별로 재구성됨.

> 평가표 페인포인트 #3(SPOF) 항목에 그대로 인용 가능한 실제 장애 사례

---

## 3. CI/CD — Shipit + Merge Queue

출처: [Automatic Deployment at Shopify](https://shopify.engineering/automatic-deployment-at-shopify), [Successfully Merging the Work of 1000+ Developers](https://shopify.engineering/successfully-merging-work-1000-developers), [Introducing the Merge Queue](https://shopify.engineering/introducing-the-merge-queue), [Software Release Culture at Shopify](https://shopify.engineering/software-release-culture-shopify)

### 자동 배포 파이프라인
```
Merge → Build container → Run CI → Ship to production
```
개발자가 GitHub에서 merge 버튼을 누르는 순간 자동으로 위 흐름이 시작됨. end-to-end 약 15분 소요를 목표로 하되 각 단계에서 취소 가능하도록 설계.

### Master를 항상 배포 가능 상태로 유지하는 3원칙
1. Master는 항상 green(CI 통과) 상태 — master가 깨지면 전체 개발이 멈춤
2. Master는 production과 크게 벌어지면 안 됨 — drift가 클수록 리스크 증가
3. 긴급 머지는 빨라야 함 — 장애 대응 시 즉시 반영 가능해야 함

### Merge Queue 동작 방식
- 개발자가 PR에 `/shipit` 코멘트 → 웹훅 발동 → Branch CI 통과 + 리뷰어 승인 확인 → 큐에 등록
- "predictive branch": PR들을 미리 병합해보는 가상 브랜치에서 CI를 실행, green인 것만 진짜 master로 병합 (master가 깨진 걸 pull 받는 상황 자체를 원천 차단)
- 오래된/많이 벌어진 브랜치는 자동 거부 (`merge.max_divergence` 설정)
- 긴급 상황엔 `/shipit --emergency`로 큐를 건너뛰고 즉시 병합 가능 (감사 로그 남음)
- GitHub Branch Protection을 프로그래밍적으로 활성화해 큐를 우회한 직접 머지를 원천 차단 — 티켓몽 Branch Protection Ruleset과 동일한 발상

### 규모
- 하루 약 400개 커밋을 master에 병합, 1,000명 이상의 개발자가 참여
- Shopify 핵심 앱 PR의 90% 이상이 Shipit+Merge Queue 경유
- CI 오케스트레이션은 Buildkite로 수백 개 워커에 병렬 분산, 40만 개 이상의 유닛 테스트를 15~20분 내로 처리

### 오픈소스
Shipit은 오픈소스([Shopify/shipit-engine](https://github.com/Shopify/shipit-engine)). MySQL/PostgreSQL/SQLite3 + Redis 필요. `shipit.yml`로 CI 필수 체크, 머지 방식(merge/squash/rebase), divergence 임계값 등을 설정.

---

## 4. Kubernetes 인프라 — 모놀리스와의 공존

출처: [Scaling Platform Engineering: Shopify's Blueprint](https://logz.io/blog/scaling-platform-engineering-shopify-blueprint/) (Logz.io 인터뷰), [Shopify Tech Stack 분석](https://blog.bytebytego.com/p/shopify-tech-stack)

### 규모
- 약 400개의 Kubernetes 클러스터 운영, 상태(DB 등)와 무상태(애플리케이션) 워크로드 모두 K8s 위에서 실행
- Black Friday 2024 기준: 하루 1,730억 요청, 분당 최대 2.84억 요청 처리
- 하루 약 1,000개 PR 병합, 프로덕션 배포는 하루 107회

### 플랫폼 조직 구조 — "platform of platforms"
인프라 그룹 산하에 데이터 플랫폼 / 관찰성 플랫폼 / 스테이트풀 시스템 / 스트리밍 플랫폼이 있고, 이들을 지탱하는 production platform이 최하단에 존재. 애플리케이션 개발자는 앱 코드만 책임지고, 플랫폼 엔지니어는 인프라를 책임지며 장애 시엔 함께 대응.

### 개발자에게 K8s를 어디까지 숨길 것인가
처음엔 K8s를 완전히 추상화한 레이어를 시도했으나 잘 작동하지 않음 → 현재는 "의미있는 기본값(golden path) + 파워유저는 직접 매니페스트 조작 가능"한 절충안으로 정착. "K8s를 개발자에게서 완전히 숨길 수는 없다"는 결론.

> 프로젝트의 GitOps 저장소 설계에도 적용 가능한 교훈: 기본 템플릿(차트)은 단순하게, 필요하면 오버라이드 가능하게 구성

### K8s 이전 역사
Kubernetes 이전엔 Chef(정적 환경에 적합한 설정관리 도구)로 배포 관리 → 동적인 관리 컨트롤 플레인 필요성 대두 → K8s(GKE) 전환.

---

## 5. 데이터·이벤트 인프라

출처: [How Shopify Manages its Petabyte Scale MySQL Database](https://blog.bytebytego.com/p/how-shopify-manages-its-petabyte)

### MySQL 백업 — RTO 단축 사례
페타바이트급 MySQL을 Pod별 샤드로 운영. 기존 백업 방식은 리전 전반에 걸쳐 시간이 오래 걸리고, 샤드당 복구 시간이 6시간 이상 소요됨. GCP Persistent Disk 스냅샷 기능을 활용해 RTO(복구 목표 시간)를 30분으로 단축. K8s CronJob으로 15분마다 전체 클러스터·리전에서 스냅샷 실행, 하루 약 100회.

> 평가표 "SLO 측정/Error Budget" 항목에 RTO 실측 수치로 인용 가능

### Kafka
메시징·이벤트 스트리밍의 중추. 프로듀서·컨슈머 디커플링, 검색·분석·비즈니스 워크플로우 지원. 피크 시 초당 6,600만 메시지 처리. 주문 생성, 상품 업데이트 같은 도메인 이벤트 발행에 사용.

---

## 6. Shopify vs Zalando — 대조 요약

| 항목 | Shopify | Zalando |
|---|---|---|
| 아키텍처 기본형 | 모듈러 모놀리스 + 격리 인프라 | 완전 MSA |
| 장애 격리 단위 | Pod (shop_id 샤딩) | Cell |
| 배포 안전장치 | Merge Queue + predictive branch | dev→stable 채널 + e2e 게이트 |
| 실제 장애 교훈 | Redismageddon → 공유 리소스 제거 | (미공개) |
| 팀 구조 | Production Engineering이 공통 인프라 담당 | 200개 이상 자율팀, "you build it, you run it" |

이 대비는 "MSA가 유일한 정답은 아니며, 모놀리스도 인프라 설계로 확장성·격리를 확보할 수 있다"는 비교분석 논점으로 활용 가능.

---

## 7. 프로젝트 적용 포인트

| 평가표 항목 | Shopify 사례 | 적용 방법 |
|---|---|---|
| 페인포인트(SPOF) | Redismageddon | 공유 리소스(단일 Redis/DB) 식별 후 서비스별 격리 설계 근거로 인용 |
| 배포 CI/CD | Merge Queue 3원칙 | "master는 항상 배포 가능해야 한다" 원칙을 팀 워크플로우에 명문화 |
| Branch Protection | 프로그래밍적 direct-merge 차단 | 티켓몽 Branch Protection Ruleset의 이론적 근거로 인용 |
| SLO/RTO 측정 | MySQL 백업 RTO 30분 단축 사례 | Error Budget 계산 시 "목표 RTO 설정 → 개선" 서술 구조 참고 |
| 장애 격리 설계 | Pod 아키텍처 | 재고/주문 서비스의 DB를 도메인별로 완전 격리하는 근거 |

---

## 8. 출처 목록

- [A Pods Architecture To Allow Shopify To Scale](https://shopify.engineering/a-pods-architecture-to-allow-shopify-to-scale)
- [E-Commerce at Scale: Inside Shopify's Tech Stack](https://shopify.engineering/e-commerce-at-scale-inside-shopifys-tech-stack)
- [Automatic Deployment at Shopify](https://shopify.engineering/automatic-deployment-at-shopify)
- [Successfully Merging the Work of 1000+ Developers](https://shopify.engineering/successfully-merging-work-1000-developers)
- [Introducing the Merge Queue](https://shopify.engineering/introducing-the-merge-queue)
- [Software Release Culture at Shopify](https://shopify.engineering/software-release-culture-shopify)
- [GitHub — Shopify/shipit-engine](https://github.com/Shopify/shipit-engine)
- [Scaling Platform Engineering: Shopify's Blueprint (Logz.io)](https://logz.io/blog/scaling-platform-engineering-shopify-blueprint/)
- [How Shopify Manages its Petabyte Scale MySQL Database (ByteByteGo)](https://blog.bytebytego.com/p/how-shopify-manages-its-petabyte)
- [Shopify Tech Stack 분석 (ByteByteGo)](https://blog.bytebytego.com/p/shopify-tech-stack)

Shopify 공식 블로그: https://shopify.engineering — "Production Engineering", "Developer Acceleration" 태그 위주로 최신 글 확인 권장
