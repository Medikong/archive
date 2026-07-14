---
id: seller-web-service-api-inventory
title: 판매자 웹 서비스 API 인벤토리
type: web-api-integration-catalog
status: draft
tags: [web, seller, api, service, openapi, code]
source: local
created: 2026-07-13
updated: 2026-07-13
---

# 판매자 웹 서비스 API 인벤토리

## 현행 Seller BFF 코드 식별자

현재 `dropmong-web`에 구현된 제거 대상 진입점과 downstream placeholder다. 이 이름은 현행 코드 식별자이며 [SD.A.20040](../../../50-service-design/A_200_seller/A_200_40-api/README.md)의 `API.A.200-*`와 별개다. 목표 구조에는 Seller BFF가 없다.

| 구분 | Method / Path | 코드 식별자 | 현재 동작 | 운영 연동 상태 |
| --- | --- | --- | --- | --- |
| seller context | `GET /api/web/seller/context` | Route Handler `GET`, `toSellerContext` | 개발용 서명 세션의 seller·membership·permission을 반환 | Auth·Seller Context 미연결 |
| onboarding | `POST /api/web/seller/onboarding` | Route Handler `POST`의 `onboarding` 분기 | 개발 모드에서 active seller fixture session으로 전환 | 운영에서는 `WEB_SELLER_CONTRACT_UNAVAILABLE` |
| browser Command | `POST /api/web/seller/{commandPath}` | `executeSellerCommand` | permission, CSRF, Origin, 멱등키, 최근 인증, 감사 IP를 검사 | downstream 미구현 |
| page 조회 대상 | `GET /web/seller/pages/{kind}` | `readSellerPage` | 개발 모드에서는 `getSellerPageFixture` 반환 | 미구현 `SELLER_MANAGEMENT_INTERNAL_BASE_URL` placeholder만 정의 |
| Command 대상 | `POST /web/seller/commands/{commandPath}` | `sendSellerCommand` | 개발 모드에서는 완료 결과 fixture 반환 | 미구현 `SELLER_MANAGEMENT_INTERNAL_BASE_URL` placeholder만 정의 |

`SELLER_CONTEXT_INTERNAL_BASE_URL`과 `SELLER_MANAGEMENT_INTERNAL_BASE_URL`은 현재 코드의 미연결 설정이며 배포 서비스를 확정한 계약이 아니다. 목표 구성에서는 둘 다 사용하지 않는다. 브라우저는 Kubernetes Ingress를 통해 실제 소유 서비스를 호출하고, 현재 BFF 설정과 client는 PAGE별 전환 뒤 제거한다.

### Page kind

| Page kind | PAGE |
| --- | --- |
| `dashboard` | `PAGE.A.200` |
| `drops` | `PAGE.A.201` |
| `products` | `PAGE.A.202` |
| `drop-editor` | `PAGE.A.203` |
| `review` | `PAGE.A.204` |
| `orders` | `PAGE.A.205` |
| `coupons` | `PAGE.A.206` |
| `analytics` | `PAGE.A.207` |
| `settlements` | `PAGE.A.208` |
| `store` | `PAGE.A.209` |
| `members` | `PAGE.A.210` |
| `issues` | `PAGE.A.211` |

### Command path

| Command path | Permission / 추가 조건 | 설계상 업무 식별자 | 상태 |
| --- | --- | --- | --- |
| `products/save` | `seller.product.write` | `CMD.A.200-06` | downstream 없음 |
| `drop-drafts/save` | `seller.drop.write` | `CMD.A.200-07` | downstream 없음 |
| `reviews/submit` | `seller.drop.review.submit` | `CMD.A.200-08` | downstream 없음 |
| `order-exports/create` | `seller.order.export`, `seller_order_export`, 감사 IP | `CMD.A.200-12` | downstream 없음 |
| `coupons/save` | `seller.coupon.write` | `API.A.19-10~13` 후보 | Coupon adapter 없음 |
| `account/save` | `seller.account.write`, `seller_account_change` | `CMD.A.200-01` | downstream 없음 |
| `store-profile/save` | `seller.store.write` | `CMD.A.200-02` | downstream 없음 |
| `members/invite` | `seller.member.write`, `seller_member_manage` | `CMD.A.200-03` | downstream 없음 |
| `roles/permissions/save` | `seller.role.permission.write`, `seller_member_manage` | `CMD.A.200-04` | downstream 없음 |
| `issues/create` | `seller.issue.write` | `CMD.A.200-15` | downstream 없음 |

`commandPath` 하나가 여러 업무 변형을 뭉뚱그리는 경우가 있다. 실제 연동에서는 `coupons/save`, `products/save`처럼 범용 이름을 그대로 downstream canonical endpoint로 확정하지 않고 각 API의 method, resource ID와 version 계약으로 분리한다.

## Auth Service

`services` 메인 `2070c20`에 OpenAPI와 실제 HTTP E2E가 존재한다.

| API ID | operationId | Method / Path | seller 사용 | 상태·추가 작업 |
| --- | --- | --- | --- | --- |
| `API.A.300-16` | `getAuthContext` | `GET /api/v1/auth/context` | session에서 인증된 `user_id`와 인증 context 확인 | 구현됨, Ingress·브라우저 계약 정합화 필요 |
| `API.A.300-17` | `reauthenticateEmail` | `POST /api/v1/auth/reauthentications/email` | 주문 자료·팀·계정 변경 전 목적 한정 proof | archive seller purpose 설계됨, 현재 서비스 bundle 구현 대기 |
| `API.A.300-29` | `resumeAuthenticatedAction` | `POST /api/v1/auth/intents/{intentId}/action-resume` | 인증 뒤 seller 작업 복귀 | archive seller action 설계됨, 현재 서비스 bundle 구현 대기 |

Auth는 seller membership이나 판매자 업무 권한을 소유하지 않는다. `API.A.300-16`의 `user_id`는 소유자가 확정된 membership API의 조회 subject로만 사용하며, 이메일·휴대폰을 seller 권한 주장에 넣지 않는다.

## Coupon Service

구현 기준 `cd60476`이 메인 `2070c20`에 병합됐다. 서비스 snapshot은 원장 `73b1e94` 계약과 일치하며, 이후 `8b3258f`에서 확정한 위험 기반 승인·발급 대기 표시·판매자 성과 범위는 서비스 후속 구현으로 남아 있다.

| API ID | operationId | Method / Path | seller 화면 후보 | 상태·제약 |
| --- | --- | --- | --- | --- |
| `API.A.19-10` | `createCouponCampaign` | `POST /api/v1/internal/coupon-campaigns` | `PAGE.A.206` 쿠폰 생성 | 메인 구현, 위험 기반 선택 승인은 후속 구현 |
| `API.A.19-11` | `configureCouponCampaignIssuance` | `PUT /api/v1/internal/coupon-campaigns/{campaignId}/issuance-policy` | 발급 수량·기간 설정 | 메인 구현, seller 범위 adapter 필요 |
| `API.A.19-12` | `reviewCouponCampaign` | `POST /api/v1/internal/coupon-campaigns/{campaignId}/reviews` | 승인·반려 상태 표시의 원천 | 메인 구현, 판매자 직접 호출 대상 아님 |
| `API.A.19-13` | `createCouponCampaignPolicyVersion` | `POST /api/v1/internal/coupon-campaigns/{campaignId}/policy-versions` | 허용된 쿠폰 수정 | 메인 구현, Ingress-facing seller version 계약 필요 |
| `API.A.19-16` | `getCouponCampaignPerformance` | `GET /api/v1/internal/coupon-campaigns/{campaignId}/performance` | `PAGE.A.206~207` 발급·사용·회수 성과 | 메인 기본 집계 구현, seller `RM.A.19-03`·`groupBy`·scope는 후속 구현 |
| `API.A.19-25` | `listCouponCostAttributions` | `GET /api/v1/internal/coupon-cost-attributions` | `PAGE.A.207~208` 할인 비용·정산 보조 | 메인 구현, seller별 집계·정산 원장은 아님 |

현재 Coupon에는 seller별 캠페인 목록 endpoint와 제휴 제안 조회·응답 endpoint가 없다. `PAGE.A.206` 전체를 `API.A.19-10~16`만으로 완성할 수 없다.

## Catalog Service

| 코드 식별자 | Method / Path | 현재 응답 | seller 사용 | 판정 |
| --- | --- | --- | --- | --- |
| `list_drops` | `GET /drops` | 공개 드롭 목록과 포함된 상품 요약 | `PAGE.A.201` 공개 상태 대조 후보 | 어댑터 필요, seller 소유·초안·검수 없음 |
| `get_drop` | `GET /drops/{drop_id}` | 공개 드롭 상세 | 승인 후 공개 결과 대조 후보 | 어댑터 필요, 다른 seller 동일 `404` 계약 없음 |

Catalog는 현재 메모리 catalog이며 seller 상품 생성·수정 API가 없다. `PAGE.A.202~204`의 원장으로 사용하지 않는다.

## Order Service

| 코드 식별자 | Method / Path | 현재 권한 | seller 사용 | 판정 |
| --- | --- | --- | --- | --- |
| `create_order` | `POST /orders` | `CUSTOMER`, `X-User-Id`, 멱등키 | seller 화면과 무관 | 사용 불가 |
| `get_order` | `GET /orders/{order_id}` | 주문 `userId`와 요청 사용자 일치 | 단건 주문 원천 참고 | 사용 불가, seller 목록·scope·마스킹 없음 |

판매자 주문 목록과 출고 자료는 Order 원장을 직접 노출하지 않고 `RM.A.200-07`과 주문 export 계약으로 제공해야 한다.

## Payment Service

| 코드 식별자 | Method / Path | 현재 용도 | seller 사용 | 판정 |
| --- | --- | --- | --- | --- |
| `approve_mock_payment` | `POST /payments/mock-approvals` | 개발용 결제 승인 | 없음 | 사용 불가 |
| `fail_mock_payment` | `POST /payments/mock-failures` | 개발용 결제 실패 | 없음 | 사용 불가 |
| `get_payment` | `GET /payments/{payment_id}` | 구매자 소유 결제 단건 | 분석 원천 참고 | 사용 불가, seller 분석 API 아님 |

브라우저와 `dropmong-web`이 Payment를 직접 조회해 매출·실패율을 계산하지 않는다. 소유자가 결정된 `RM.A.200-08` 집계가 Event로 재구성 가능한 값을 제공해야 한다.

## Notification Service

| 코드 식별자 | Method / Path | 현재 권한 | seller 사용 | 판정 |
| --- | --- | --- | --- | --- |
| `list_notifications` | `GET /notifications` | `CUSTOMER` 사용자 알림 | 상단 seller 알림 후보가 아님 | 사용 불가 |

seller 알림 또는 운영 이슈 갱신 알림을 제공하려면 seller principal과 알림 분류 계약을 별도로 설계한다. 현재 `PAGE.A.211`의 원천은 `RM.A.200-10`이다.

## 논리 판매자 계약의 MSA 배치

| 필요한 경계 | 논리 API | 목표 소유 서비스 | 현재 상태 |
| --- | --- | --- | --- |
| Seller Access·Management | `API.A.200-01~08` | 미확정 | 보호 route 공개 불가 |
| Seller Proposal | `API.A.200-09~16` | `catalog-service` 후보 | seller 계약 미구현 |
| Seller Dashboard·Analytics | `API.A.200-17`, `API.A.200-23~24` | 미확정 | 외부 Event와 조회 모델 소유자 필요 |
| Seller Drop | `API.A.200-18` | `catalog-service` 후보 | seller 계약 미구현 |
| Seller Order Export | `API.A.200-19~22` | `order-service` 후보 | seller 주문·export 계약 미구현 |
| Seller Issue | `API.A.200-25~28` | 미확정 | 플랫폼 운영 결과 계약 대기 |
| Seller Partnership | 제휴 제안 조회·응답 | `coupon-service` 검토 대상 | canonical operation 없음 |

method/path, request/response, 오류, 멱등·ETag는 [operation catalog](../../../50-service-design/A_200_seller/A_200_40-api/operation-catalog.md), 외부 Event gap은 [Event 계약](../../../50-service-design/A_200_seller/A_200_40-api/event-contracts.md)을 따른다.
