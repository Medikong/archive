---
id: project-design-template
title: 프로젝트 설계 템플릿
type: project-template
status: active
tags: [project-design, ddd, ui, backend, template]
source: local
created: 2026-07-06
updated: 2026-07-07
---

# 프로젝트 설계 템플릿

이 폴더는 프론트엔드 구현 여부와 관계없이 프로젝트를 A-to-Z로 설계하기 위한 작업용 템플릿이다. 핵심 원칙은 요구사항, 사이트맵, UI, 유스케이스, 이벤트스토밍/바운디드 컨텍스트, 도메인 모델, 영속성 설계, 서비스 인덱스, API, 처리 시나리오를 모두 식별자와 Markdown 링크로 연결하는 것이다.

## 사용 순서

1. `00-requirements/.template/requirements.md`로 요구사항, 제약, 제외 범위를 먼저 정리한다.
2. `10-sitemap/README.md`에 전체 사이트맵을 flowchart로 먼저 그린다.
3. `10-sitemap/.template/PAGE_A_XX.md`를 복사해 페이지 문서를 만든다.
4. `20-ui/.template/ui-asset.md`로 이미지, 문서, 와이어프레임 같은 UI 근거를 정리한다.
5. `30-uc/README.md`에서 유스케이스 작성 컨텍스트를 확인하고 `30-uc/INDEX.md`에 유스케이스 문서 목록을 관리한다.
6. `30-uc/.template/use-case.md`로 사용자 관점 작업 단위를 만든다.
7. `40-event-storming-bounded-context/.template/bounded-context.md`로 이벤트스토밍 결과와 도메인 경계를 정리한다.
8. `50-domain-model/.template/aggregate-entity.md`로 Aggregate와 Entity를 한 문서에 설계한다.
9. `55-persistence/.template/persistence-design.md`로 DB 스키마, Repository 근거, 읽기/쓰기 전략을 정리한다.
10. `60-service/.template/service-index.md`로 구현과 운영 단위에서 필요한 설계 문서를 모아 본다.
11. `70-api/.template/api-endpoint-process.md`로 API 엔드포인트와 API 내부 처리 과정을 한 문서에 설계한다.
12. `80-scenario/.template/scenario.md`로 특정 화면 행동이나 API 호출에서 발생 가능한 처리 상황을 정리한다.

## 폴더 번호 규칙

최상위 설계 폴더는 10단위 번호를 사용한다. 중간 단계가 추가되면 기존 폴더 번호와 링크를 흔들지 않고 `15-*`, `25-*`처럼 끼워 넣는다.

```text
00-requirements/
10-sitemap/
20-ui/
30-uc/
40-event-storming-bounded-context/
50-domain-model/
55-persistence/
60-service/
70-api/
80-scenario/
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
| `BC.A.01` | `BC_A_01_order.md` |

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

## 예제 세트

주문 결제 예제는 요구사항부터 시나리오까지 같은 식별자 묶음으로 연결되어 있다. 처음 볼 때는 각 폴더의 README를 기준으로 [요구사항](00-requirements/README.md), [사이트맵](10-sitemap/README.md), [UI](20-ui/README.md), [유스케이스 작성 컨텍스트](30-uc/README.md), [유스케이스 인덱스](30-uc/INDEX.md), [이벤트스토밍과 BC](40-event-storming-bounded-context/README.md), [도메인 모델](50-domain-model/README.md), [영속성 설계](55-persistence/README.md), [서비스](60-service/README.md), [API](70-api/README.md), [시나리오](80-scenario/README.md) 순서로 보면 된다.

샘플 문서는 [REQ.A.01](00-requirements/.example/REQ_A_01_order_checkout.md), [PAGE.A.01](10-sitemap/.examples/PAGE_A_01_order_checkout.md), [UI.A.01](20-ui/.examples/UI_A_01_order_checkout_wireframe.md), [UC.A.01](30-uc/.examples/UC_A_01_place_order.md), [BC.A.01](40-event-storming-bounded-context/.examples/BC_A_01_order.md), [AGG.A.01](50-domain-model/.examples/AGG_A_01_order.md), [PST.A.01](55-persistence/.examples/PST_A_01_order_persistence.md), [SVC.A.01](60-service/.examples/SVC_A_01_order_service.md), [API.A.01](70-api/.examples/API_A_01_place_order.md), [SCN.A.01](80-scenario/.examples/SCN_A_01_place_order.md)로 이어진다.

## 예시와 실제 작업

각 폴더의 `.examples/`는 템플릿 사용법을 보여주는 샘플 보관소다. 실제 프로젝트 설계 문서는 `.examples/`가 아니라 각 설계 유형 폴더의 루트에 만든다.

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

🏷️ 플로우 참조: FLOW.A.01 | 페이지 참조: [PAGE.A.01](10-sitemap/.examples/PAGE_A_01_order_checkout.md) | UI 참조: [UI.A.01](20-ui/.examples/UI_A_01_order_checkout_wireframe.md) | 영속성 참조: [PST.A.01](55-persistence/.examples/PST_A_01_order_persistence.md) | 서비스 참조: [SVC.A.01](60-service/.examples/SVC_A_01_order_service.md) | 시나리오 참조: [SCN.A.01](80-scenario/.examples/SCN_A_01_place_order.md) | API 참조: [API.A.01](70-api/.examples/API_A_01_place_order.md)
```

## 폴더

각 문서 폴더의 `README.md`는 해당 설계 유형의 기본 진입점이다. 폴더에 별도 `INDEX.md`가 있으면 실제 문서 목록은 `INDEX.md`에서 관리한다.

- [00-requirements](00-requirements/README.md): 요구사항, 사용자, 제약, 제외 범위, 수용 기준.
- [10-sitemap](10-sitemap/README.md): 전체 사이트맵 인덱스와 페이지 중심 허브 문서.
- [20-ui](20-ui/README.md): 스크린샷, 이미지, 문서, 와이어프레임 같은 UI 근거.
- [30-uc](30-uc/README.md): 유스케이스 작성 컨텍스트. 실제 유스케이스 문서 목록은 [INDEX](30-uc/INDEX.md)에서 관리한다.
- [40-event-storming-bounded-context](40-event-storming-bounded-context/README.md): 이벤트스토밍 결과, 도메인 언어, 바운디드 컨텍스트 경계.
- [50-domain-model](50-domain-model/README.md): Aggregate와 Entity를 함께 설명하는 도메인 모델.
- [55-persistence](55-persistence/README.md): 데이터베이스 스키마, Repository 설계 근거, 읽기/쓰기 전략.
- [60-service](60-service/README.md): API, 도메인 모델, 바운디드 컨텍스트, Kubernetes, 관측성 설계를 모아 보는 서비스 인덱스.
- [70-api](70-api/README.md): API 엔드포인트와 API 내부 처리 과정.
- [80-scenario](80-scenario/README.md): UI 행동이나 API 호출에서 발생 가능한 처리 상황.

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
- Aggregate/Entity는 UI 필드 목록이 아니라 도메인 책임과 불변조건 기준으로 모델링한다.
- 영속성 설계는 도메인 모델을 어떤 저장 구조와 Repository 전략으로 구현할지 기록한다.
- 서비스 문서는 설계를 새로 정의하지 않고 API, 도메인 모델, BC, 운영 설계를 찾아가기 위한 인덱스로만 둔다.
- API는 화면 호출 경로와 도메인 Command/Query 사이의 계약으로 둔다.
- 불확실한 필드는 `확인 필요`, 추정한 필드는 `추정`으로 표시한다.
