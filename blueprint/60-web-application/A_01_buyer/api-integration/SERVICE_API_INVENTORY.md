---
id: buyer-web-service-api-inventory
title: 구매자 웹 서비스 API 인벤토리
type: web-api-integration-catalog
status: draft
tags: [web, buyer, api, service, openapi, code]
source: local
created: 2026-07-16
updated: 2026-07-16
---

# 구매자 웹 서비스 API 인벤토리

## 현재 웹 진입점

| 화면 책임 | 서버 함수 | Route Handler | 호출 방식 | 현재 원천 |
| --- | --- | --- | --- | --- |
| 홈 | `getHomePage` | `GET /api/web/home` | Server Component는 함수 직접 호출 | Catalog 설정 또는 mock |
| 상품 상세 | `getProductDetailPage` | `GET /api/web/products/[productId]` | Server Component는 함수 직접 호출 | Catalog 설정 또는 mock |
| Auth context | `getServerActor`, `getRequestActor` | `GET /api/web/auth/context` | 개발 cookie 직접 해석 | 개발 mock |
| Checkout snapshot | `getCheckoutSnapshot` | `GET /api/web/checkouts/[checkoutId]` | Server Component는 함수 직접 호출 | fixture |
| Checkout confirm | `confirmCheckout` | `POST /api/web/checkouts/[checkoutId]/confirm` | 브라우저 Route Handler 호출 | fixture |
| 주문 결과 | `getOrderResult` | `GET /api/web/orders/[orderId]` | 최초 서버 함수, 이후 browser polling | fixture |

Route Handler가 있다는 사실은 downstream 서비스 연결을 뜻하지 않는다.

## 서비스별 상태

| 서비스·책임 | 실제 API | 서비스 API | 웹 코드 | 프론트 | 로컬·검증 | GitOps·Ingress | 다음 판단 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `catalog-service` | `GET /drops`, `GET /drops/{dropId}` | 구현 | `catalog.ts` 지원 | URL 설정 시 가능, 현재 기본은 mock | 로컬·CI·Docker URL 없음 | 선언 없음 | 첫 실연동 대상 |
| `auth-service` | `GET /api/v1/auth/context`, 로그인·session API | 구현 | 실제 client 없음 | 개발 cookie | mock만 | HEAD의 ticketing/MediKong 표기 `/auth`는 현재 작업 트리에서 비활성, canonical 경로·DropMong 동일성·실제 배포 미확인 | browser session 계약 정합화 후 연결 |
| Checkout | canonical snapshot·confirm 없음 | 소유 미확정 | fixture 함수·handler | fixture | mock만 | 소유·선언 미확정 | 소유자·계약 먼저 결정 |
| `order-service` | `POST /orders`, `GET /orders/{orderId}` | 구현 | client·URL 없음 | `dev-order.*` fixture | mock만 | 선언 없음 | Checkout 뒤 주문 상태 계약 정합화 |
| `payment-service` | mock 승인·실패, `GET /payments/{paymentId}` | 구현 | client·URL 없음 | fixture 결제수단·결과 | mock만 | ticketing/MediKong 표기의 동명 `/payments` 선언, DropMong 동일성 미확인 | Checkout 뒤 결제 상태 계약 정합화 |
| `coupon-service` | claim, code redemption, 보유 쿠폰 목록·상세 | 구현 | client 없음 | Checkout 문구·할인 fixture | 없음 | DropMong private-dev·`/coupons` 선언은 있으나 canonical `/api/v1/**`와 불일치, 실제 배포 미확인 | 쿠폰 지갑과 Checkout 계약 분리 |
| `interest-service` | 관심 추가·삭제·목록, 예정·인기 랭킹 | 구현 | client 없음 | 홈·상세 개인화 문구만 | 없음 | 선언 없음 | PAGE.A.09·22·23 연결 후보 |
| `user-service` | 사용자 생성, 본인 프로필 조회·수정 | 구현 | client 없음 | 미연결 | 없음 | DropMong private-dev와 exact `/api/v1/users`, Prefix `/api/v1/users/me` 선언, 실제 배포 미확인 | PAGE.A.10 프로필 축 후보 |
| `notification-service` | `GET /notifications` | 구현 | client 없음 | 미연결 | 없음 | ticketing/MediKong 표기의 동명 `/notifications` 선언, DropMong 동일성 미확인 | buyer 알림 section 후보 |
| 장바구니 | canonical API 없음 | 소유 미확정 | route 없음 | 미연결 | 없음 | 소유·선언 미확정 | 원장 소유자 결정 |
| 배송 | canonical 조회·관리 API 없음 | 소유 미확정 | route 없음 | 주문 완료 문구 fixture | 없음 | 소유·선언 미확정 | 조회·관리 소유자 결정 |
| 포인트 | 서비스 inventory에 원장 없음 | 소유 미확정 | client 없음 | Checkout 문구·금액 fixture | 없음 | 소유·선언 미확정 | 원장 소유자 결정 |
| 결제수단 | canonical 조회 API 없음 | 소유 미확정 | client 없음 | `MOCK_CARD` fixture | 없음 | 소유·선언 미확정 | 결제·Checkout 책임 결정 |

## Catalog

- `listDrops`는 `CATALOG_INTERNAL_BASE_URL`이 있으면 `/drops?limit=100`을 호출한다.
- `getProductWithDrop`은 `dropId`와 base URL이 있으면 `/drops/{dropId}`를 먼저 호출한다.
- base URL이 없고 `DEV_MOCK_MODE=true`일 때만 `developmentDrops`를 사용한다.
- upstream 연결 실패, timeout, non-2xx와 schema 오류를 mock으로 바꾸지 않는다.
- `.env.local`, CI, Playwright와 Docker smoke에는 Catalog URL이 없다.

판정: `서비스 API 구현 / 웹 코드 지원 / 로컬 mock / CI 실연동 미검증 / GitOps 선언 없음`.

## Auth

- `auth-service`에 `GET /api/v1/auth/context` route, OpenAPI와 HTTP E2E가 있다.
- 현재 service 구현은 mobile credential 경계를 사용하므로 browser opaque session 계약을 추가로 맞춰야 한다.
- `dropmong-web`의 `auth.ts`는 `dropmong_dev_session`을 직접 해석하고 실제 Auth를 호출하지 않는다.
- mock-off에서는 `WEB_AUTH_CONTRACT_UNAVAILABLE`로 실패한다.

판정: `서비스 API 구현 / browser 계약 보강 필요 / 웹 client 없음 / 개발 cookie / 동명 배포 선언의 DropMong 동일성 미확인`.

## Checkout·Order·Payment

- `getCheckoutSnapshot`은 배송지, 배송비, `MOCK_CARD`, 쿠폰·포인트 문구와 총액을 fixture로 만든다.
- `confirmCheckout`은 실제 Order·Payment를 호출하지 않고 `dev-order.*`를 만든다.
- `getOrderResult`은 주문 완료, 결제 금액과 예상 배송일을 fixture로 반환한다.
- Order·Payment 서비스 API가 존재해도 웹 설정과 client에는 두 서비스가 없다.
- BFF가 두 API를 임의 순서로 호출하는 방식으로 Checkout 소유권을 대신하지 않는다.

## 부가 Query

- Coupon: `/api/v1/users/me/coupons`와 단건 조회가 있어 PAGE.A.19 후보지만 웹 미연결이다.
- Interest: 관심 목록·변경과 예정·인기 랭킹이 있어 PAGE.A.09·22·23 후보지만 웹 미연결이다.
- User: 본인 프로필이 있어 PAGE.A.10의 필수 프로필 축 후보지만 웹 미연결이다.
- Notification: 구매자 알림 목록이 있지만 웹 미연결이다.
- 마이 화면이 여러 section을 보여 주더라도 각 원장의 오류와 최신성을 유지하고 가짜 `0`이나 빈 목록으로 바꾸지 않는다.

## 설정·배포 판정

| 근거 | 확인 결과 |
| --- | --- |
| `.env.local` | `DEV_MOCK_MODE=true`, Catalog URL 주석 |
| `.env.example` | mock 기본, Catalog URL 예시만 주석 |
| GitHub Actions | mock true, Catalog·Auth·Order·Payment URL 없음 |
| Playwright | mock true, buyer fixture 구매 검증 |
| Docker smoke | mock true, health·ready·metrics·로그만 확인 |
| `config/services.yml` | 9개 image 목록, 배포 증거 아님 |
| GitOps - Web·Catalog | `dropmong-web`, `catalog-service`와 Catalog 환경 변수 선언 없음 |
| GitOps - User | DropMong private-dev와 `/api/v1/users`, `/api/v1/users/me` Ingress 선언 있음. 실제 동기화 미확인 |
| GitOps - Coupon | DropMong private-dev와 `/coupons` Ingress 선언 있음. canonical `/api/v1/**`와 경로 불일치, 실제 동기화 미확인 |
| GitOps - Auth·Payment·Notification | ticketing/MediKong 표기의 동명 선언만 확인. Auth HEAD `/auth`는 기존 작업 트리에서 비활성이고 canonical 경로를 포함하지 않음. 현재 DropMong checkout과의 동일성 미확인 |
| GitOps - Order·Interest | 해당 서비스 선언 없음 |

라이브 클러스터는 미확인이다. 따라서 `선언 있음`, `route 포함`, `실제 배포`를 같은 상태로 합치지 않는다.
