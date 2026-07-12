---
id: project-design-template
title: 프로젝트 설계 템플릿
type: project-template
status: active
tags: [project-design, ddd, ui, backend, template]
source: local
created: 2026-07-06
updated: 2026-07-10
---

# 프로젝트 설계 템플릿

이 폴더는 프론트엔드 구현 여부와 관계없이 프로젝트를 A-to-Z로 설계하기 위한 작업용 템플릿이다. 핵심 원칙은 요구사항, 사이트맵, UI, 유스케이스, 이벤트스토밍/바운디드 컨텍스트, 바운디드 컨텍스트별 서비스 상세 설계, 처리 시퀀스를 모두 식별자와 Markdown 링크로 연결하는 것이다.

## 사용 순서

1. `00-requirements/.template/requirements.md`로 요구사항, 제약, 제외 범위를 먼저 정리한다.
2. `10-sitemap/README.md`에 전체 사이트맵을 flowchart로 먼저 그린다.
3. `10-sitemap/.template/PAGE_A_XX.md`를 복사해 페이지 문서를 만든다.
4. `20-ui/.template/ui-asset.md`로 이미지, 문서, 와이어프레임 같은 UI 근거를 정리한다.
5. `30-uc/README.md`에서 유스케이스 작성 컨텍스트를 확인하고 `30-uc/INDEX.md`에 유스케이스 문서 목록을 관리한다.
6. `30-uc/.template/use-case.md`로 사용자 관점 작업 단위를 만든다.
7. `40-event-storming-bounded-context/.template/bounded-context.md`로 이벤트스토밍 결과와 도메인 경계를 정리한다.
8. `50-service-design/<context>/README.md`를 만들고 하나의 바운디드 컨텍스트 또는 서비스 단위로 상세 설계 범위를 잡는다.
9. `50-service-design/<context>/<context_id>_10-domain-model/`에서 Aggregate, Entity, Value Object, Policy, Business Rule을 설계한다.
10. `50-service-design/<context>/<context_id>_20-persistence/`에서 Repository, DB 스키마, 인덱스, 마이그레이션 전략을 정리한다.
11. `50-service-design/<context>/<context_id>_30-service/`에서 Application Service, Domain Service, Command Handler, 트랜잭션 경계를 설계한다.
12. `50-service-design/<context>/<context_id>_40-api/`에서 API 엔드포인트, 요청/응답, 오류, 이벤트 계약을 설계한다.
13. `60-web-application/README.md`에서 반응형 웹, 상태·데이터, 애플리케이션 수준 BFF, 배포·관측성 기준을 정리한다.
14. `80-sequence/.template/scenario.md`로 여러 화면 행동, API 호출과 Context 간 처리 순서를 정리한다.

## 폴더 번호 규칙

최상위 설계 폴더는 10단위 번호를 사용한다. 중간 단계가 추가되면 기존 폴더 번호와 링크를 흔들지 않고 `15-*`, `25-*`처럼 끼워 넣는다.

```text
00-requirements/
10-sitemap/
20-ui/
30-uc/
40-event-storming-bounded-context/
50-service-design/
60-web-application/
80-sequence/
```

`50-service-design/` 안에서는 컨텍스트 식별자 뒤에 `10`, `20`, `30`, `40` 하위 번호를 붙인다. 설계 분야 하나가 파일 하나로 끝나지 않을 수 있으므로 각 분야를 파일이 아니라 폴더로 둔다.

```text
50-service-design/
  A_300_auth/
    A_300_10-domain-model/
    A_300_20-persistence/
    A_300_30-service/
    A_300_40-api/
```

## 피처 그룹 규칙

식별자 가운데의 알파벳은 피처 그룹을 뜻한다. `A`는 첫 번째 피처 묶음, `B`는 두 번째 피처 묶음처럼 사용한다. 같은 피처 안에서는 `01`, `02`, `03` 순서로 문서를 추가한다.

| 그룹 | 의미 | 예시 |
| --- | --- | --- |
| `A` | 주문 결제 예제 피처 | `PAGE.A.01`, `SCN.A.01` |
| `B` | 다음 피처 묶음 | `PAGE.B.01`, `SCN.B.01` |

## 식별자 규칙

| Prefix | 의미 | 예시 |
| --- | --- | --- |
| `REQ` | 요구사항 | `REQ.A.01` |
| `PP` | 요구사항 문서 하위 페인포인트 | `REQ.A.01.PP.01` |
| `FP` | 요구사항 문서 하위 기능 포인트 | `REQ.A.01.FP.01` |
| `NFP` | 요구사항 문서 하위 비기능 포인트 | `REQ.A.01.NFP.01` |
| `FLOW` | 유저 플로우 단계 | `FLOW.A.01` |
| `PAGE` | 사이트맵의 단일 페이지 또는 페이지 그룹 | `PAGE.A.01` |
| `UI` | 이미지, 문서, 와이어프레임 등 UI 에셋 | `UI.A.01` |
| `UC` | 페이지/기능 기반 유스케이스 | `UC.A.01` |
| `SCN` | 처리 시나리오 | `SCN.A.01` |
| `BC` | Bounded Context | `BC.A.01` |
| `SD` | Service Design | `SD.A.300`, `SD.A.30010` |
| `AGG` | Aggregate | `AGG.A.01` |
| `ENT` | Entity | `ENT.A.01` |
| `VO` | Value Object | `VO.A.01` |
| `RM` | Read Model | `RM.A.01` |
| `CMD` | Command | `CMD.A.01` |
| `QRY` | Query | `QRY.A.01` |
| `EVT` | Event | `EVT.A.01` |
| `PST` | Persistence Design | `PST.A.01` |
| `SVC` | Service Index | `SVC.A.01` |
| `API` | API Endpoint | `API.A.01` |
| `WEB` | Web Application Design | `WEB.A.01` |
| `BFF` | Browser-facing Application Module | `BFF.A.01` |
| `ERR` | Error Case | `ERR.A.03` |

## 파일명 규칙

문서 안 식별자는 점 표기, 파일명은 언더스코어 표기를 사용한다.

| 식별자 | 파일명 예시 |
| --- | --- |
| `REQ.A.01` | `REQ_A_01_order_checkout.md` |
| `PAGE.A.01` | `PAGE_A_01_order_checkout.md` |
| `UI.A.01` | `UI_A_01_order_checkout_wireframe.md` |
| `UC.A.01` | `UC_A_01_place_order.md` |
| `SCN.A.01` | `SCN_A_01_place_order.md` |
| `AGG.A.01` | `AGG_A_01_order.md` |
| `PST.A.01` | `PST_A_01_order_persistence.md` |
| `SVC.A.01` | `SVC_A_01_order_service.md` |
| `API.A.01` | `API_A_01_place_order.md` |
| `WEB.A.01` | `WEB_A_01_frontend_architecture.md` |
| `BFF.A.01` | `BFF_A_01_web_bff_module.md` |
| `BC.A.01` | `BC_A_01_limited_drop_commerce.md` |
| `BC.EX.01` | `BC_EX_01_order.md` (예제 전용) |

## 요구사항 하위 식별자 규칙

요구사항 문서 안의 페인포인트, 기능 포인트, 비기능 포인트는 전역 단독 ID를 쓰지 않고 요구사항 문서 ID를 접두사로 붙인다. `FP.01`처럼 쓰면 어떤 요구사항 문서의 항목인지 구분하기 어렵기 때문이다.

| 유형 | 의미 | 예시 |
| --- | --- | --- |
| `PP` | Pain Point | `REQ.A.01.PP.01` |
| `FP` | Functional Point | `REQ.A.01.FP.01` |
| `NFP` | Non-Functional Point | `REQ.A.01.NFP.01` |

## 페이지 문서 단위 규칙

`PAGE` 문서는 반드시 물리적인 화면 하나만 뜻하지 않는다. 사용자 목적, 시나리오, 또는 flow가 하나로 묶이는 경우에는 페이지 그룹 폴더 안에 여러 페이지 문서를 함께 둘 수 있다.

- 단일 페이지가 독립적인 목적과 상태를 가지면 문서 하나에 페이지 하나를 둔다.
- 여러 화면이 하나의 사용자 목적을 공유하면 `PAGE_A_300_auth_member/`처럼 그룹 시작 ID와 그룹 이름으로 폴더를 만들고, 대표 문서는 `PAGE_A_300_auth_member.md`처럼 하나로 관리한다.
- 대표 문서의 front matter에는 `page_ids`로 포함 화면을 적고, 본문에서는 `PAGE.A.300`, `PAGE.A.301`처럼 고유한 `PAGE` ID를 유지한다.
- 페이지 그룹의 포함 페이지 목록과 대표 flow는 해당 그룹의 시작 페이지 문서나 폴더 README 중 하나에 적는다.
- 요구사항, UI, 유스케이스, API, 시나리오는 문서 파일명이 아니라 필요한 `PAGE` ID를 기준으로 참조한다.
- 상태 전이, redirect, 인증/권한, 실패 처리처럼 화면 사이 관계가 중요한 내용은 페이지 그룹 폴더 안에서 서로 링크해 설명한다.
- 같은 페이지 그룹에 대응하는 UI 문서는 `UI_A_300_auth_member/`처럼 같은 그룹 이름의 UI 폴더를 만들고, 대표 문서 `UI_A_300_auth_member.md` 안에서 포함 `UI` ID와 에셋을 함께 관리한다.

## 클라이언트 폴더 규칙

구매자, 판매자, 운영자처럼 사용 목적과 화면 구성이 다른 클라이언트는 `10-sitemap/`과 `20-ui/` 아래에 각각 전용 진입 폴더를 둔다. PAGE와 UI의 클라이언트 폴더 이름을 맞추면 같은 화면군을 양쪽에서 바로 찾을 수 있다.

```text
10-sitemap/
  buyer-mobile-web/
    README.md
    PAGE_A_01_homepage.md
20-ui/
  buyer-mobile-web/
    README.md
    UI_A_01_homepage.md
    assets/
      UI_A_01_homepage/
```

- 클라이언트 폴더의 `README.md`는 해당 클라이언트의 페이지 목록, URL, 우선 사용자 여정, 반응형 기준을 관리한다.
- UI 에셋은 UI 문서와 같은 클라이언트 폴더의 `assets/<UI 문서 ID>/`에 둔다.
- 인증처럼 여러 클라이언트가 실제로 공유하는 페이지 그룹은 공용 위치에 유지하고, 화면이나 배포가 갈라질 때 클라이언트 폴더로 나눈다.
- 클라이언트 폴더는 배포 단위를 자동으로 뜻하지 않는다. 하나의 웹 애플리케이션 안에서도 route group과 레이아웃 경계를 드러내기 위해 사용할 수 있다.

## 예제 세트

주문 결제 예제는 요구사항부터 시나리오까지 같은 `A.01` 식별자 묶음으로 연결한다. 다만 BC 예제는 실제 `BC.A.01`과 충돌하지 않도록 예제 전용 `BC.EX.01`을 사용한다. 처음 볼 때는 각 폴더의 README를 기준으로 [요구사항](00-requirements/README.md), [사이트맵](10-sitemap/README.md), [UI](20-ui/README.md), [유스케이스 작성 컨텍스트](30-uc/README.md), [유스케이스 인덱스](30-uc/INDEX.md), [이벤트스토밍과 BC](40-event-storming-bounded-context/README.md), [서비스 상세 설계](50-service-design/README.md), [웹 애플리케이션 설계](60-web-application/README.md), [처리 시퀀스](80-sequence/README.md) 순서로 보면 된다.

샘플 문서는 [REQ.A.01](00-requirements/.example/REQ_A_01_order_checkout.md), [PAGE.A.01](10-sitemap/.examples/PAGE_A_01_order_checkout.md), [UI.A.01](20-ui/.examples/UI_A_01_order_checkout_wireframe.md), [UC.A.01](30-uc/.examples/UC_A_01_place_order.md), [BC.EX.01](40-event-storming-bounded-context/.examples/BC_EX_01_order.md), [AGG.A.01](50-service-design/.examples/order/A_01_10-domain-model/AGG_A_01_order.md), [PST.A.01](50-service-design/.examples/order/A_01_20-persistence/PST_A_01_order_persistence.md), [SVC.A.01](50-service-design/.examples/order/A_01_30-service/SVC_A_01_order_service.md), [API.A.01](50-service-design/.examples/order/A_01_40-api/API_A_01_place_order.md), [SCN.A.01](80-sequence/.examples/SCN_A_01_place_order.md)로 이어진다.

## 예시와 실제 작업

각 폴더의 `.examples/`는 템플릿 사용법을 보여주는 샘플 보관소다. 실제 프로젝트 설계 문서는 `.examples/`가 아니라 각 설계 유형 폴더의 루트에 만든다. `50-service-design/`은 컨텍스트 폴더 아래에 실제 설계 문서를 둔다. `60-web-application/`은 도메인 서비스가 아니라 브라우저에 제공하는 웹 애플리케이션과 같은 배포 단위의 BFF 모듈을 다룬다.

```text
20-ui/
├── .examples/
│   └── UI_A_01_order_checkout_wireframe.md
├── .template/
│   └── ui-asset.md
├── README.md
└── UI_B_01_actual_feature_wireframe.md
```

## 링크 방식

긴 추적성 테이블 하나에 모든 정보를 억지로 넣지 않는다. 각 문서의 `연관 태그` 섹션에 참조 유형을 라벨로 붙이고 Markdown 링크를 나열한다.

```markdown
## 연관 태그

🏷️ 플로우 참조: FLOW.A.01 | 페이지 참조: [PAGE.A.01](10-sitemap/.examples/PAGE_A_01_order_checkout.md) | UI 참조: [UI.A.01](20-ui/.examples/UI_A_01_order_checkout_wireframe.md) | 도메인 참조: [AGG.A.01](50-service-design/.examples/order/A_01_10-domain-model/AGG_A_01_order.md) | 영속성 참조: [PST.A.01](50-service-design/.examples/order/A_01_20-persistence/PST_A_01_order_persistence.md) | 서비스 참조: [SVC.A.01](50-service-design/.examples/order/A_01_30-service/SVC_A_01_order_service.md) | API 참조: [API.A.01](50-service-design/.examples/order/A_01_40-api/API_A_01_place_order.md) | 시나리오 참조: [SCN.A.01](80-sequence/.examples/SCN_A_01_place_order.md)
```

## 폴더

각 문서 폴더의 `README.md`는 해당 설계 유형의 기본 진입점이다. 폴더에 별도 `INDEX.md`가 있으면 실제 문서 목록은 `INDEX.md`에서 관리한다.

- [00-requirements](00-requirements/README.md): 요구사항, 사용자, 제약, 제외 범위, 수용 기준.
- [10-sitemap](10-sitemap/README.md): 전체 사이트맵 인덱스와 페이지 중심 허브 문서.
- [20-ui](20-ui/README.md): 스크린샷, 이미지, 문서, 와이어프레임 같은 UI 근거.
- [30-uc](30-uc/README.md): 유스케이스 작성 컨텍스트. 실제 유스케이스 문서 목록은 [INDEX](30-uc/INDEX.md)에서 관리한다.
- [40-event-storming-bounded-context](40-event-storming-bounded-context/README.md): 이벤트스토밍 결과, 도메인 언어, 바운디드 컨텍스트 경계.
- [50-service-design](50-service-design/README.md): 바운디드 컨텍스트별 도메인 모델, 영속성, 서비스, API 상세 설계.
- [60-web-application](60-web-application/README.md): 반응형 웹 구조, 상태·데이터 전략, 애플리케이션 수준 BFF, 배포·관측성·테스트.
- [80-sequence](80-sequence/README.md): 여러 UI 행동, API 호출과 Context가 참여하는 처리 시퀀스.

## 에셋 규칙

에셋은 단일 루트 폴더에 모으지 않고, 해당 에셋을 소유하는 설계 유형 폴더 안에 둔다. 같은 유형 안에서도 문서 ID별 하위 폴더를 둬서 어떤 문서의 에셋인지 추적한다.

```text
20-ui/
├── UI_B_01_actual_feature_wireframe.md
└── assets/
    └── UI_B_01_actual_feature_wireframe/
        └── actual-feature-wireframe.svg
```

- 에셋 소유자는 해당 에셋을 만들거나 갱신하는 문서다.
- 다른 설계 문서가 같은 에셋을 써야 하면 소유 폴더의 에셋을 상대 경로로 참조한다.
- 여러 문서가 공통으로 쓰는 에셋도 먼저 나온 소유 문서를 기준으로 두고, 필요하면 나중에 별도 공용 폴더를 추가한다.
- Mermaid 원본과 렌더링 결과를 함께 둘 때는 같은 문서 ID 폴더 안에 `.mmd`와 `.svg`를 같이 둔다.

## 작성 원칙

- 요구사항 문서는 구현 목록이 아니라 사용자의 문제, 목표, 제약, 판단 기준을 먼저 적는다.
- 페이지 문서는 사용자에게 보이는 것과 사용자가 할 수 있는 행동을 먼저 적는다.
- 페이지 그룹 폴더는 포함 페이지 목록, 대표 사용자 목적, 페이지 사이 이동과 상태 전이를 찾기 쉽게 둔다.
- 페이지 문서의 상태와 이동 가능 페이지는 `연관 사이트맵` flowchart의 노드와 edge label로 표현한다.
- 전역 내비게이션은 앱 공통 구조로 보고, 개별 페이지 문서에는 사용자 목적에 직접 필요한 진입, 복귀, 상태 전이만 적는다.
- UI 문서에는 실제 에셋 경로와 캡처 기준을 남긴다.
- 시나리오는 화면 이벤트나 API 호출 같은 시작점에서 처리 과정을 설명한다.
- `50-service-design`은 이벤트스토밍에서 분해한 바운디드 컨텍스트 단위로 작성한다.
- Aggregate/Entity는 UI 필드 목록이 아니라 도메인 책임과 불변조건 기준으로 모델링한다.
- 영속성 설계는 도메인 모델을 어떤 저장 구조와 Repository 전략으로 구현할지 기록한다.
- 서비스 설계는 Application Service, Domain Service, Command Handler, 트랜잭션 경계를 명확히 한다.
- API는 화면 호출 경로와 도메인 Command/Query 사이의 계약으로 둔다.
- 웹 애플리케이션 설계는 PAGE/UI를 실제 route와 컴포넌트로 연결하되 도메인 규칙을 다시 소유하지 않는다.
- BFF는 웹 애플리케이션과 같은 배포 단위의 모듈로 시작하고, 세션·CSRF·화면 DTO·호출 조정만 맡는다.
- 불확실한 필드는 `확인 필요`, 추정한 필드는 `추정`으로 표시한다.
