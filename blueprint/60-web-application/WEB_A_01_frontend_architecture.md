---
id: WEB.A.01
title: 프론트엔드 애플리케이션 구조
type: web-application-architecture
status: draft
tags: [web-application, frontend, nextjs, app-router, route-group, responsive]
source: local
created: 2026-07-10
updated: 2026-07-10
---

# 프론트엔드 애플리케이션 구조

## 기본 정보

- Web Application ID: `WEB.A.01`
- 프레임워크: Next.js App Router, React, TypeScript
- 배포 기준: 하나의 반응형 웹 애플리케이션
- 사용자 영역: 구매자, 판매자, 플랫폼 운영자, 인증
- 상태와 데이터 기준: [WEB.A.02](WEB_A_02_state_data_strategy.md)
- 웹 BFF 경계: [BFF.A.01](BFF_A_01_web_bff_module.md)
- 배포와 검증 기준: [WEB.A.03](WEB_A_03_deployment_observability_test.md)

## 설계 목표

- KT 클라우드 네이티브 실무 프로젝트에서 구매, 판매 준비, 운영 대응을 실제 화면으로 검증할 수 있게 한다.
- 초기 구현은 코드베이스와 배포 단위를 하나로 유지해 빌드, 배포, 공통 인증, 관찰 지점을 단순하게 가져간다.
- 하나의 앱 안에서도 액터별 URL, layout, 권한, 컴포넌트, 오류 영향을 구분해 이후 분리 비용을 낮춘다.
- 초기 HTML과 읽기 화면은 Server Component를 우선하고, 브라우저 상호작용은 작은 Client Component로 제한한다.

## 기준 결정

1. 구매자, 판매자, 플랫폼 운영자 화면은 하나의 Next.js App Router 애플리케이션으로 시작한다.
2. 최상위 `app/layout.tsx`는 문서 셸과 공통 provider만 소유한다. 액터별 내비게이션과 접근 확인은 `(buyer)`, `(seller)`, `(operator)`, `(auth)` route group의 layout이 소유한다.
3. route group 이름은 URL에 포함하지 않는다. 실제 URL은 사이트맵의 `path`를 따른다.
4. 페이지와 layout은 Server Component를 기본값으로 사용한다. 상호작용, 브라우저 API, polling이 필요한 leaf component만 Client Component로 만든다.
5. 마이크로 프론트엔드와 액터별 별도 저장소는 초기 범위에서 사용하지 않는다.

## PAGE와 UI 연결

| Route group | URL 예시 | PAGE 기준 | UI 기준 | 기본 layout |
| --- | --- | --- | --- | --- |
| `(buyer)` | `/`, `/products/{productId}` | [PAGE.A.01](../10-sitemap/buyer-mobile-web/PAGE_A_01_homepage.md), [PAGE.A.02](../10-sitemap/buyer-mobile-web/PAGE_A_02_product_detail.md) | [UI.A.01](../20-ui/buyer-mobile-web/UI_A_01_homepage.md), [UI.A.02](../20-ui/buyer-mobile-web/UI_A_02_product_detail.md) | 구매자 상단 앱 바와 하단 내비게이션 |
| `(buyer)` | `/cart`, `/checkout` | [PAGE.A.06](../10-sitemap/buyer-mobile-web/PAGE_A_06_shopping_cart.md), [PAGE.A.11](../10-sitemap/buyer-mobile-web/PAGE_A_11_payment.md) | [UI.A.06](../20-ui/buyer-mobile-web/UI_A_06_shopping_cart.md), [UI.A.11](../20-ui/buyer-mobile-web/UI_A_11_payment.md) | 구매자 셸, 결제 집중 layout은 하단 내비게이션을 생략할 수 있음 |
| `(buyer)` | `/orders/complete`, `/orders`, `/orders/{orderId}/tracking`, `/orders/{orderId}/manage` | [PAGE.A.14](../10-sitemap/buyer-mobile-web/PAGE_A_14_order_complete.md), [PAGE.A.15](../10-sitemap/buyer-mobile-web/PAGE_A_15_order_history.md), [PAGE.A.16](../10-sitemap/buyer-mobile-web/PAGE_A_16_track_order.md), [PAGE.A.17](../10-sitemap/buyer-mobile-web/PAGE_A_17_shipping_order_manage.md) | [UI.A.14](../20-ui/buyer-mobile-web/UI_A_14_order_complete.md), [UI.A.15](../20-ui/buyer-mobile-web/UI_A_15_order_history.md), [UI.A.16](../20-ui/buyer-mobile-web/UI_A_16_track_order.md), [UI.A.17](../20-ui/buyer-mobile-web/UI_A_17_shipping_order_manage.md) | 구매자 주문 layout |
| `(buyer)` | `/my`, `/my/coupons` | [PAGE.A.10](../10-sitemap/buyer-mobile-web/PAGE_A_10_my.md), [PAGE.A.19](../10-sitemap/buyer-mobile-web/PAGE_A_19_coupon_wallet/PAGE_A_19_owned_coupon.md) | [UI.A.10](../20-ui/buyer-mobile-web/UI_A_10_my.md), [UI.A.19](../20-ui/buyer-mobile-web/UI_A_19_coupon_wallet/UI_A_19_coupon_wallet.md) | 로그인 구매자 layout |
| `(auth)` | `/auth/signin`, `/auth/signup`, `/auth/password-reset` | [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md), [PAGE.A.310](../10-sitemap/PAGE_A_310_password_find/PAGE_A_310_password_find.md) | [UI.A.300](../20-ui/UI_A_300_auth_member/UI_A_300_auth_member.md), [UI.A.310](../20-ui/UI_A_310_password_find/UI_A_310_password_find.md) | 인증 전용 layout |
| `(seller)` | `/seller`, `/seller/drops`, `/seller/orders` | [PAGE.A.200~211](../10-sitemap/PAGE_A_200_seller_portal/README.md) | [UI.A.200~211](../20-ui/UI_A_200_seller_portal/README.md) | 판매자 데스크톱 셸 |
| `(operator)` | `/operator/*` | [UC.A.03](../30-uc/UC_A_03_platform_operator_admin.md)의 페이지 문서 작성 후 연결 | UI 문서 작성 후 연결 | 플랫폼 운영자 셸 |

플랫폼 운영자 route group은 코드 위치만 예약한다. `PAGE`와 `UI`가 정의되기 전에는 임의의 운영 메뉴를 구현하지 않는다.

## 폴더 구조 예시

```text
src/
├── app/
│   ├── layout.tsx
│   ├── global-error.tsx
│   ├── not-found.tsx
│   ├── (buyer)/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── products/[productId]/
│   │   │   ├── page.tsx
│   │   │   ├── loading.tsx
│   │   │   ├── error.tsx
│   │   │   ├── not-found.tsx
│   │   │   └── _components/
│   │   ├── cart/page.tsx
│   │   ├── checkout/
│   │   │   ├── page.tsx
│   │   │   ├── loading.tsx
│   │   │   └── error.tsx
│   │   ├── orders/
│   │   │   ├── page.tsx
│   │   │   ├── complete/page.tsx
│   │   │   └── [orderId]/
│   │   │       ├── tracking/page.tsx
│   │   │       └── manage/page.tsx
│   │   └── my/
│   │       ├── page.tsx
│   │       └── coupons/page.tsx
│   ├── (auth)/
│   │   ├── layout.tsx
│   │   └── auth/
│   │       ├── signin/page.tsx
│   │       ├── signup/page.tsx
│   │       └── password-reset/page.tsx
│   ├── (seller)/
│   │   ├── layout.tsx
│   │   └── seller/
│   │       ├── page.tsx
│   │       ├── drops/page.tsx
│   │       ├── drops/[dropId]/edit/page.tsx
│   │       └── orders/page.tsx
│   └── (operator)/
│       ├── layout.tsx
│       └── operator/
│           └── page.tsx
├── components/
│   ├── shared/
│   ├── buyer/
│   ├── seller/
│   └── operator/
├── features/
│   ├── auth/
│   ├── drop/
│   ├── checkout/
│   └── order-status/
└── lib/
    ├── api/
    ├── auth/
    ├── query/
    └── observability/
```

## 폴더 책임

| 위치 | 책임 | 금지 사항 |
| --- | --- | --- |
| `app/**/page.tsx` | route 진입, 서버 조회 시작, 화면 조합 | 대형 Client Component, 업무 규칙 재구현 |
| `app/**/layout.tsx` | 액터별 셸, 내비게이션, 공통 접근 확인 | layout 확인만으로 API 권한이 보장된다고 가정 |
| `app/**/_components` | 해당 route에서만 사용하는 화면 조각 | 다른 액터 route에서 직접 import |
| `components/shared` | 표현과 접근성 규칙이 같은 공통 컴포넌트 | 구매자와 운영자의 업무 의미를 하나로 합치기 |
| `features/*` | 페이지 여러 곳에서 재사용하는 사용자 행동과 화면 조합 | 백엔드 도메인 모델과 영속성 소유 |
| `lib/api` | 타입이 있는 API client, 오류 변환, trace header 전달 | React state와 UI 메시지 소유 |
| `lib/auth` | session 확인, 로그인 복귀 위치, 권한 확인 helper | 브라우저 store에 session 원문 저장 |

공용 `index.ts`를 통한 일괄 export는 만들지 않는다. route와 feature는 필요한 모듈을 실제 파일에서 직접 import해 의존 관계와 번들 경계를 드러낸다.

## Server Component와 Client Component

| 판단 | Server Component | Client Component |
| --- | --- | --- |
| 기본 사용처 | page, layout, 읽기 중심 섹션 | 버튼, 폼 상호작용, modal, 브라우저 API |
| 데이터 | 초기 조회, session 기반 조회, 캐시 가능한 공개 데이터 | polling, 입력 중 상태, 즉시 사용자 피드백 |
| 보안 | cookie와 server-only credential 사용 | 민감 credential을 prop이나 browser storage로 받지 않음 |
| 렌더링 | HTML과 RSC payload를 만들고 필요한 데이터만 전달 | 전달받은 최소 필드로 상호작용 처리 |

- 서버에서 독립적으로 조회할 데이터는 `Promise.all` 또는 독립된 async component로 동시에 시작한다.
- 느린 섹션은 전체 page를 막지 않도록 의미 있는 `Suspense` 경계 안에서 렌더링한다.
- Client Component에는 전체 API DTO를 넘기지 않고 실제 상호작용에 필요한 필드만 전달한다.
- countdown은 Client Component가 화면 숫자만 갱신한다. 드롭 오픈 여부와 구매 가능 판정은 서버 응답을 따른다.
- Server Action을 사용하더라도 각 action 안에서 입력 검증, 인증, 권한을 다시 확인한다. layout이나 middleware 통과만 신뢰하지 않는다.
- 주문, 결제처럼 중복 실행 위험이 있는 mutation은 API 계약의 멱등키를 반드시 전달한다.

## Layout 기준

| Layout | 포함 요소 | 접근 기준 |
| --- | --- | --- |
| root | `<html>`, `<body>`, 전역 스타일, 최소 provider, 전역 오류 추적 | 모든 사용자 |
| buyer | 구매자 헤더, 하단 내비게이션, 반응형 content container | 공개 화면과 로그인 화면을 segment에서 구분 |
| auth | 인증 제목, 단계 안내, 로그인 후 복귀 정보 | 인증 전후 모두 가능 |
| seller | 좌측 메뉴, 판매자 식별 정보, 작업 알림 영역 | 판매자 role 필요 |
| operator | 운영 메뉴, 환경 표시, 위험 작업 경고 | 플랫폼 운영 role 필요 |

최상위 root layout은 하나만 둔다. route group마다 별도 root layout을 만들지 않아 액터 영역을 이동할 때 전체 문서가 다시 로드되는 일을 피한다.

## 로딩, 오류, 빈 결과

| 상태 | 구현 위치 | 사용자 표시 |
| --- | --- | --- |
| route 로딩 | 데이터 대기 범위의 `loading.tsx` 또는 `Suspense` fallback | 실제 UI 크기와 비슷한 skeleton, 진행 중인 대상 |
| route 오류 | 가까운 segment의 `error.tsx` | 재시도, 안전한 이전 위치, 추적 ID |
| 전역 오류 | `global-error.tsx` | 최소 복구 화면과 홈 이동 |
| 리소스 없음 | `not-found.tsx` 또는 `notFound()` | 잘못된 ID와 비공개/삭제 상태를 정책에 맞게 구분 |
| 빈 결과 | page 또는 feature의 `EmptyState` | 정상 조회 결과임을 알리고 다음 행동 제공 |
| 최신성 지연 | page 또는 feature의 `StaleState` | 서버 기준 최신 시각, 지연 안내, 새로고침 |

빈 목록은 예외가 아니므로 `error.tsx`에서 처리하지 않는다. 결제 진행 중, 품절, 주문 조회 지연처럼 업무 의미가 있는 상태는 관련 `PAGE`, `UI`, 오류 계약에 맞춰 feature component가 표시한다.

`error.tsx`와 `global-error.tsx`는 오류 경계와 재시도 동작이 필요하므로 Client Component로 작성한다. 오류 화면이 전체 page를 Client Component로 바꾸는 근거가 되지는 않는다.

## 반응형 기준

- 구매자 영역은 작은 화면을 먼저 설계하고 넓은 화면에서는 콘텐츠 폭과 열 수를 늘린다.
- 판매자와 플랫폼 운영자 영역은 넓은 화면의 표와 작업 패널을 기준으로 하되, 작은 화면에서는 핵심 필드와 주요 작업을 카드 또는 단계형 화면으로 바꾼다.
- 작은 화면에서 표를 단순히 가로 축소하지 않는다. 업무 우선순위가 낮은 열은 상세 화면으로 옮긴다.
- route group별 layout이 다른 탐색 구조를 가져도 공통 focus, keyboard, 오류 알림 규칙은 유지한다.
- UI 에셋과 구현 결과가 다르면 먼저 `UI` 문서의 반응형 의도를 확인하고, 구현에서 임의로 의미를 바꾸지 않는다.

## API 접근 경계

- 화면 DTO, 웹 session, CSRF, 내부 요청 변환의 상세 기준은 [BFF.A.01](BFF_A_01_web_bff_module.md)이 소유한다.
- 초기 Server Component 조회는 서버 전용 API client를 사용한다.
- 브라우저에서 주기적으로 갱신해야 하는 주문, 결제, 배송 상태는 브라우저에 노출 가능한 동일 출처 endpoint를 사용한다.
- 내부 서비스 주소, 서비스 credential, 운영용 endpoint는 Client Component에 노출하지 않는다.
- 웹 전용 응답 조합이 필요해도 백엔드 업무 규칙을 프론트엔드에 복제하지 않는다. 조합 계약의 소유 위치는 서비스/API 설계에서 정한다.
- API 오류는 `lib/api`에서 기술 오류와 업무 오류로 구분하고, `PAGE`와 `UI`가 요구하는 사용자 메시지로 연결한다.

## 애플리케이션 분리 기준

다음 신호가 반복해서 확인되기 전에는 하나의 앱을 유지한다.

| 분리 신호 | 확인 근거 |
| --- | --- |
| 배포 주기와 승인 절차가 독립적임 | 액터 영역 하나의 변경 때문에 전체 앱 배포가 지연되는 기록 |
| 보안과 네트워크 경계가 다름 | 운영자 앱의 사설 접근, 별도 WAF, 별도 인증 체계 요구 |
| 장애 영향과 SLO가 다름 | 판매자/운영자 코드 장애가 구매자 구매 지표에 영향을 준 측정 결과 |
| 번들 크기와 초기 성능이 악화됨 | route 단위 lazy loading 후에도 액터별 코드가 핵심 Web Vitals를 지속적으로 저해 |
| 팀이 계약 없이 같은 코드를 자주 충돌함 | 소유권, 테스트, 배포 기록에서 반복되는 병목 |
| UI 셸과 기술 요구가 양립하기 어려움 | 운영 대시보드와 구매자 화면이 공통 runtime 제약을 서로 방해 |

분리할 때는 기존 route group을 별도 앱의 시작점으로 옮기고 `PAGE`, `UI`, API 계약은 그대로 유지한다. 초기부터 마이크로 프론트엔드 runtime을 추가하지 않는다.

## 검증 기준

- 모든 공개 route가 하나 이상의 `PAGE`와 `UI` 문서에 연결된다.
- 각 page와 layout은 Server Component가 기본이며 `'use client'` 범위가 leaf component에 한정된다.
- 구매자, 판매자, 운영자 layout과 권한 확인이 서로 분리된다.
- 로딩, 오류, 빈 결과, 최신성 지연을 각각 재현할 수 있다.
- 작은 화면과 넓은 화면에서 핵심 CTA와 상태가 가려지지 않는다.
- 주문과 결제 mutation은 중복 제출을 막고 멱등키를 전달한다.
- 앱을 나누지 않은 이유와 향후 분리 신호를 배포·보안·성능 지표로 설명할 수 있다.

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.01](../00-requirements/REQ_A_01_limited_drop_commerce.md), [REQ.A.03](../00-requirements/REQ_A_03_seller.md), [REQ.A.04](../00-requirements/REQ_A_04_platform_operator_admin.md), [REQ.A.05](../00-requirements/REQ_A_05_auth_member.md) | 페이지 참조: [사이트맵 인덱스](../10-sitemap/README.md) | UI 참조: [UI 인덱스](../20-ui/README.md) | UC 참조: [UC.A.01](../30-uc/UC_A_01_buyer_purchase_delivery.md), [UC.A.02](../30-uc/UC_A_02_seller_manage_drop.md), [UC.A.03](../30-uc/UC_A_03_platform_operator_admin.md), [UC.A.300](../30-uc/UC_A_300_auth_member.md) | BFF 참조: [BFF.A.01](BFF_A_01_web_bff_module.md) | 서비스 참조: [서비스 상세 설계](../50-service-design/README.md) | 시나리오 참조: [처리 시퀀스](../80-sequence/README.md) | 배포·검증 참조: [WEB.A.03](WEB_A_03_deployment_observability_test.md)

## 확인 필요

- 플랫폼 운영자 `PAGE`와 `UI`가 작성된 뒤 `(operator)`의 실제 route 목록을 확정한다.
- 구매자 checkout을 buyer layout 안에서 유지할지, 탐색 요소를 줄인 별도 nested layout으로 둘지 정한다.
- Server Component가 호출할 웹 전용 API 진입점과 브라우저 polling endpoint의 소유 서비스를 정한다.
- 배포 환경의 Next.js 서버 인스턴스 수와 캐시 공유 방식을 [WEB.A.02](WEB_A_02_state_data_strategy.md)에 맞춰 확정한다.

## 참고 자료

- [Next.js App Router](https://nextjs.org/docs/app)
- [Next.js 프로젝트 구조와 route group](https://nextjs.org/docs/app/getting-started/project-structure)
- [Next.js Server Component와 Client Component](https://nextjs.org/docs/app/getting-started/server-and-client-components)
