---
id: REQ.A.08
title: 반응형 웹 애플리케이션 요구사항 정의
type: requirements
status: draft
tags: [requirements, dropmong, web-application, responsive-web, bff, cloud-native]
source: local
created: 2026-07-10
updated: 2026-07-10
---

# 반응형 웹 애플리케이션 요구사항 정의

## 기본 정보

- Requirements ID: `REQ.A.08`
- 프로젝트: DropMong
- 작성 기준일: 2026-07-10
- 주요 사용자: 비회원, 구매자, 판매자, 플랫폼 운영자, CS 담당자
- 주요 이해관계자: 프론트엔드 담당자, 인증 및 도메인 API 담당자, 플랫폼/SRE 담당자, QA 담당자
- 설계 범위: 반응형 웹 제공 방식, 공개/보호 영역 경계, 서버 세션, 애플리케이션 수준 BFF, 접근성, 성능, 브라우저 호환성, Kubernetes 배포와 클라우드 네이티브 시연 기준
- 제외 범위: 기존 페이지별 기능 재정의, iOS/Android 네이티브 앱, PWA 설치·오프라인·서비스 워커·웹 푸시, 도메인 데이터 소유, 범용 API Gateway 대체

## 구조 분석

### 문서 역할

- [사이트맵 인덱스](../10-sitemap/README.md), [UI 인덱스](../20-ui/README.md), [유스케이스 인덱스](../30-uc/INDEX.md)는 화면별 기능과 사용자의 작업을 정의하는 원장이다.
- 이 문서는 기존 화면 기능을 반복하지 않고, 모든 웹 화면에 공통으로 적용할 실행 환경과 품질 기준을 정의한다.
- [REQ.A.05](REQ_A_05_auth_member.md)는 인증 식별자, 세션 수명, 권한과 로그인 결과를 소유한다. 이 문서는 브라우저가 그 결과를 안전하게 사용하는 서버 세션과 로그인 게이트를 정의한다.
- [REQ.A.06](REQ_A_06_kubernetes_cluster_architecture.md)는 클러스터와 GitOps 기반을 소유하고, [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md)은 웹 애플리케이션의 배포·관측성·테스트 구현 기준을 구체화한다.

### 애플리케이션 수준 BFF 판단

- 브라우저는 도메인 서비스 주소와 내부 자격 증명을 직접 다루지 않고 같은 출처의 웹 애플리케이션 서버를 호출한다.
- BFF는 Next.js 애플리케이션과 같은 배포 단위에서 서버 세션 확인, 화면용 응답 조합, 오류 정규화, 요청 식별자와 trace context 전파를 담당한다.
- BFF는 별도 바운디드 컨텍스트나 범용 게이트웨이가 아니며 재고, 주문, 결제, 쿠폰, 인증의 업무 규칙과 원장을 소유하지 않는다.
- 단일 도메인 API 호출을 불필요하게 재구현하지 않고, 화면에 필요한 조합이나 브라우저 보안 경계가 있을 때만 중계한다.

## 문제 정의

- 사용자가 겪는 문제: 모바일과 데스크톱에서 같은 서비스를 이용해도 화면 크기, 로그인 상태, 부분 장애에 따라 탐색과 작업 결과가 일관되지 않으면 구매와 운영 작업을 신뢰하기 어렵다.
- 현재 방식의 한계: 사이트맵과 UI 시안만으로는 실제 브라우저에서 세션, 서비스 호출, 오류, 성능, 접근성, 배포 상태가 어떻게 연결되는지 검증할 수 없다.
- 이 프로젝트가 해결해야 하는 것: DropMong은 하나의 반응형 웹으로 핵심 사용자 시나리오를 실행하고, 서버 세션과 애플리케이션 수준 BFF를 통해 도메인 서비스 경계를 노출하지 않으며, 배포·관측·롤백 결과를 시연할 수 있어야 한다.
- 해결하지 않는 것: 네이티브 앱과 PWA 기능을 동시에 만들지 않는다. 웹 계층이 도메인 서비스를 대신해 업무 상태를 확정하거나 별도 업무 DB를 소유하지 않는다.

## 목표

- 사용자 목표: 사용자는 모바일과 데스크톱에서 동일한 정보와 작업 결과를 확인하고, 필요한 시점에 로그인한 뒤 원래 위치로 돌아간다.
- 비즈니스 목표: 기존 요구사항과 UI 자산을 실제로 동작하는 최소 웹 제품으로 연결해 DropMong의 핵심 시나리오를 설명한다.
- 운영 목표: 운영자는 웹 버전, 오류율, 지연, Web Vitals, 배포 상태를 확인하고 카나리 중단과 롤백 여부를 판단한다.
- 기술 목표: 브라우저 자격 증명을 서버 세션으로 보호하고, BFF와 하위 서비스 사이의 요청을 하나의 trace로 연결하며, 같은 산출물을 환경별 설정으로 배포한다.

## 사용자 유형

| Actor ID | 사용자 | 목표 | 주요 행동 |
| --- | --- | --- | --- |
| `ACTOR-GUEST` | 비회원 | 로그인 없이 공개 정보를 탐색한다. | 공개 페이지 접근, 보호 작업 시 로그인 진입 |
| `ACTOR-BUYER` | 구매자 | 인증 상태를 유지하며 구매 관련 화면을 사용한다. | 로그인, 보호 페이지 접근, 결과 확인 |
| `ACTOR-SELLER` | 판매자 | 허용된 판매자 웹 영역을 사용한다. | 서버 세션 확인, 역할 기반 접근 |
| `ACTOR-PLATFORM-OPERATOR` | 플랫폼 운영자·CS 담당자 | 높은 권한이 필요한 웹 영역을 안전하게 사용한다. | 강한 인증, 역할 확인, 감사 가능한 작업 |
| `ACTOR-ONCALL` | 플랫폼/SRE 담당자 | 웹 애플리케이션의 배포와 품질을 확인한다. | 지표·로그·trace 확인, 카나리 판정, 롤백 |

## 기능 요구사항

| Req ID | 요구사항 | 사용자 | 우선순위 | 연결 Page/UC |
| --- | --- | --- | --- | --- |
| `REQ.A.08.FR-001` | 시스템은 모바일, 태블릿, 데스크톱에서 같은 URL 체계와 기능 원장을 사용하는 반응형 웹을 제공한다. | 전체 사용자 | Must | [사이트맵](../10-sitemap/README.md), [UI](../20-ui/README.md) |
| `REQ.A.08.FR-002` | 시스템은 공개 탐색으로 분류된 페이지와 데이터에 로그인하지 않은 사용자의 접근을 허용한다. | 비회원, 구매자 | Must | [REQ.A.05](REQ_A_05_auth_member.md) |
| `REQ.A.08.FR-003` | 시스템은 개인 정보, 결제, 구매 시도, 판매자, 운영자 영역에 서버 측 로그인 게이트를 적용하고 안전한 내부 복귀 위치를 보존한다. | 비회원, 구매자, 판매자, 플랫폼 운영자 | Must | [REQ.A.05](REQ_A_05_auth_member.md) |
| `REQ.A.08.FR-004` | 웹 애플리케이션은 HttpOnly cookie 기반 서버 세션을 사용하고 브라우저 저장소에 access token이나 refresh token을 보관하지 않는다. | 인증 사용자 | Must | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |
| `REQ.A.08.FR-005` | 브라우저의 업무 API 요청은 같은 출처의 애플리케이션 수준 BFF를 거쳐 내부 서비스로 전달되며, BFF는 세션을 검증한 최소 사용자 context만 전달한다. | 전체 사용자 | Must | [SD.A.30040](../50-service-design/A_300_auth/A_300_40-api/README.md) |
| `REQ.A.08.FR-006` | BFF는 화면용 조회 조합, 입력 검증, 오류 정규화와 timeout을 담당하되 도메인 상태를 확정하거나 업무 원장을 저장하지 않는다. | 시스템 | Must | [서비스 설계](../50-service-design/README.md) |
| `REQ.A.08.FR-007` | 시스템은 구매자, 판매자, 플랫폼 운영자와 CS 역할을 서버에서 확인하고 허용되지 않은 메뉴, 페이지와 작업을 거부한다. | 인증 사용자 | Must | [REQ.A.03](REQ_A_03_seller.md), [REQ.A.04](REQ_A_04_platform_operator_admin.md) |
| `REQ.A.08.FR-008` | 공개 페이지의 선택적 개인화 영역이나 일부 하위 서비스가 실패하면 사용 가능한 공개 영역은 유지하고, 실패한 영역에는 최신성과 다음 행동을 설명하는 제한 상태를 표시한다. | 비회원, 구매자 | Must | [REQ.A.05](REQ_A_05_auth_member.md) |
| `REQ.A.08.FR-009` | 웹 애플리케이션은 요청마다 `X-Request-Id`를 확정하고 W3C trace context를 BFF와 하위 서비스에 전달하며 사용자 오류 화면에는 공유 가능한 요청 식별자를 제공한다. | 전체 사용자, 온콜 | Must | [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md) |
| `REQ.A.08.FR-010` | 시스템은 API 주소, 공개 origin, 기능 플래그, telemetry 설정을 환경별 런타임 설정으로 주입하고 이미지 재빌드 없이 배포 환경을 바꿀 수 있게 한다. | 플랫폼/SRE 담당자 | Must | [REQ.A.06](REQ_A_06_kubernetes_cluster_architecture.md) |
| `REQ.A.08.FR-011` | 로그인 만료, 갱신 실패와 로그아웃은 서버 세션을 기준으로 일관되게 처리하고, 보호 요청을 성공처럼 보이게 대체하지 않는다. | 인증 사용자 | Must | [REQ.A.05](REQ_A_05_auth_member.md) |
| `REQ.A.08.FR-012` | 시스템은 동의된 범위와 설정된 표본 비율에 따라 Web Vitals와 클라이언트 오류를 수집하며 cookie, token, 이메일, 휴대폰, 입력 본문을 telemetry에 포함하지 않는다. | 전체 사용자, 온콜 | Must | [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md) |

## 비기능 요구사항

| Req ID | 요구사항 | 기준 | 연결 문서 |
| --- | --- | --- | --- |
| `REQ.A.08.NFR-001` | 반응형 레이아웃은 작은 모바일부터 데스크톱까지 가로 스크롤 없이 핵심 콘텐츠와 작업을 제공해야 한다. | 최소 검증 viewport는 `360x800`, `390x844`, `768x1024`, `1440x900`이며 표처럼 넓은 업무 데이터만 명시적인 내부 스크롤을 허용한다. | [UI 인덱스](../20-ui/README.md) |
| `REQ.A.08.NFR-002` | 주요 브라우저에서 핵심 시나리오가 동등하게 동작해야 한다. | Chrome, Edge, Firefox, Safari의 최신 두 개 주요 안정 버전과 최신·직전 iOS Safari/Android Chrome을 지원 대상으로 둔다. | [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md) |
| `REQ.A.08.NFR-003` | 웹은 WCAG 2.2 AA를 목표로 접근 가능해야 한다. | 키보드만으로 작업 가능, 보이는 focus, 의미 있는 landmark/label, 색상 외 상태 표시, 오류와 수정 방법, `prefers-reduced-motion` 대응을 검증한다. | [UI 인덱스](../20-ui/README.md) |
| `REQ.A.08.NFR-004` | 실제 사용자 성능은 Core Web Vitals의 good 기준을 목표로 해야 한다. | p75 기준 LCP `2.5s` 이하, INP `200ms` 이하, CLS `0.1` 이하를 모바일과 데스크톱에서 각각 측정한다. | [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md) |
| `REQ.A.08.NFR-005` | 애플리케이션 수준 BFF가 추가하는 지연을 분리해 측정해야 한다. | 단일 하위 호출에서 BFF 자체 처리 p95는 하위 서비스 대기 시간을 제외하고 `150ms` 이하를 초기 목표로 둔다. | [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md) |
| `REQ.A.08.NFR-006` | 서버 세션과 변경 요청은 브라우저 공격에 안전해야 한다. | cookie에 `Secure`, `HttpOnly`, `SameSite`, `Path=/`를 적용하고 unsafe method는 CSRF와 Origin을 검증하며 외부 redirect를 허용하지 않는다. | [REQ.A.05](REQ_A_05_auth_member.md) |
| `REQ.A.08.NFR-007` | 브라우저와 서버 산출물에 비밀 값이 포함되지 않아야 한다. | `NEXT_PUBLIC_*`, HTML, JavaScript bundle, source map, 로그에 session secret, 내부 token, 개인 정보를 포함하지 않는다. | [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md) |
| `REQ.A.08.NFR-008` | 일부 서비스 장애가 웹 전체 장애로 확대되지 않아야 한다. | timeout, 요청 취소, 오류 경계와 제한 상태를 사용하고 성공처럼 보이는 빈 값이나 오래된 값을 출처 표시 없이 사용하지 않는다. | [REQ.A.01](REQ_A_01_limited_drop_commerce.md) |
| `REQ.A.08.NFR-009` | 웹 요청은 버전과 환경까지 포함해 관측 가능해야 한다. | route template, status, duration, request id, trace id, app version, environment를 지표·구조화 로그·span에서 연결한다. | [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md) |
| `REQ.A.08.NFR-010` | 웹 배포는 무중단과 빠른 롤백을 지원해야 한다. | health probe, 최소 가용 replica, graceful shutdown, 이전 API와 호환되는 카나리, 자동·수동 롤백 기준을 둔다. | [REQ.A.06](REQ_A_06_kubernetes_cluster_architecture.md), [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md) |
| `REQ.A.08.NFR-011` | 정적 자산과 공개 응답은 배포 버전이 섞이지 않게 캐시해야 한다. | 해시된 정적 자산은 immutable cache를 사용하고 HTML·세션 응답은 사용자별 cache 금지와 적절한 `Vary`/`Cache-Control`을 적용한다. | [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md) |
| `REQ.A.08.NFR-012` | 자동화 테스트는 모바일과 데스크톱의 인증·권한·장애 상태를 함께 검증해야 한다. | Playwright 프로젝트별 핵심 시나리오, 접근성 검사, 실패 시 trace와 screenshot을 CI 증거로 남긴다. | [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md) |

## 제약 조건

- 정책/법적 제약: 개인정보, cookie, telemetry 수집은 최소 수집·목적 제한·보관 기간 기준을 따르고 민감 정보는 브라우저 관측 데이터에 포함하지 않는다.
- 기술 제약: 웹 애플리케이션과 BFF는 하나의 배포 단위로 시작하며, BFF 전용 DB와 도메인 이벤트 원장을 추가하지 않는다.
- 일정 제약: KT 클라우드 네이티브 실무 프로젝트에서는 핵심 사용자 시나리오와 배포·관측·롤백 시연을 우선하고 전체 사이트맵 구현을 완료 조건으로 두지 않는다.
- 데이터 제약: BFF는 하위 서비스의 응답 최신성과 오류를 보존하며 임의의 성공 응답이나 업무 상태를 만들지 않는다.
- 운영 제약: 네이티브 앱과 PWA는 MVP에서 제외하고, 브라우저 지원 범위와 Web Vitals 표본 비율은 릴리스마다 기록한다.

## 수용 기준

- `360x800`, `390x844`, `768x1024`, `1440x900`에서 선택한 핵심 시나리오가 가로 페이지 스크롤 없이 완료된다.
- 비회원은 공개 탐색 영역을 이용할 수 있고, 보호 페이지나 작업은 로그인으로 이동한 뒤 검증된 내부 위치로 복귀한다.
- 인증 cookie는 JavaScript에서 읽을 수 없고 access/refresh token은 localStorage, sessionStorage, IndexedDB에 저장되지 않는다.
- 브라우저 업무 API 호출은 같은 출처의 BFF를 거치며, BFF는 별도 업무 DB 없이 하위 서비스의 계약과 오류를 보존한다.
- 판매자와 플랫폼 운영자는 서버에서 확인된 역할에 허용된 영역만 접근한다.
- 키보드 탐색, focus, label, 대비, 오류 안내와 자동 접근성 검사가 핵심 페이지에서 통과한다.
- 모바일·데스크톱 p75 Web Vitals와 BFF 추가 지연을 대시보드에서 버전별로 확인할 수 있다.
- 브라우저 요청 하나를 BFF와 최소 하나의 하위 서비스 span, 구조화 로그, 요청 식별자로 연결할 수 있다.
- ConfigMap과 Secret 변경이 이미지 재빌드 없이 환경별 배포에 반영되고 필수 비밀 값이 없으면 서버가 명확히 시작 실패한다.
- Playwright가 공개 접근, 로그인 게이트, 서버 세션, 역할 거부, 부분 장애를 모바일·데스크톱에서 검증한다.
- 카나리 버전의 오류율이나 지연 기준을 의도적으로 위반해 중단 또는 롤백하고, 안정 버전 복구를 지표와 화면에서 확인한다.
- manifest, service worker와 오프라인 cache 없이 일반 반응형 웹으로 제공되어 PWA가 MVP에 포함되지 않았음을 확인한다.

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.01](REQ_A_01_limited_drop_commerce.md), [REQ.A.03](REQ_A_03_seller.md), [REQ.A.04](REQ_A_04_platform_operator_admin.md), [REQ.A.05](REQ_A_05_auth_member.md), [REQ.A.06](REQ_A_06_kubernetes_cluster_architecture.md) | 페이지 참조: [사이트맵 인덱스](../10-sitemap/README.md) | UI 참조: [UI 인덱스](../20-ui/README.md) | UC 참조: [유스케이스 인덱스](../30-uc/INDEX.md) | 서비스 참조: [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) | API 참조: [SD.A.30040](../50-service-design/A_300_auth/A_300_40-api/README.md) | 웹 애플리케이션 참조: [WEB.A.03](../60-web-application/WEB_A_03_deployment_observability_test.md)

## 열린 질문

- 구매자, 판매자, 플랫폼 운영자 웹을 하나의 Next.js 애플리케이션과 배포 단위로 유지할지, 독립적인 릴리스 주기가 필요해질 때 분리할지 결정한다.
- 서버 세션의 실제 저장 위치, idle/absolute TTL과 다중 기기 정책은 Auth 설계의 어떤 artifact를 기준으로 확정할 것인가?
- BFF 조회 조합이 커질 때 구매자와 운영자 모듈을 같은 프로세스에 유지할 수 있는 분리 기준은 무엇인가?
- CSP report-only 적용 기간과 보고 endpoint를 어디에 둘 것인가?
- Web Vitals와 클라이언트 오류의 환경별 표본 비율과 보관 기간을 어떻게 정할 것인가?

## 확인 필요

- 실제 웹 저장소와 Next.js/Node.js 기준 버전, 패키지 관리자, 이미지 저장소
- `local`, `private-dev`, `aws-dev`의 public origin, 내부 서비스 주소와 TLS 인증서
- 지원 브라우저의 실제 사용자 비중과 추가 viewport 요구
- Auth server session 검증 API와 BFF internal context 계약
- 초기 부하 테스트에서 측정한 BFF 자체 지연, Web Vitals, replica당 동시 요청 수
- CI에서 사용할 Playwright browser image와 접근성 검사 도구 버전
