---
id: seller-web-service-api-inventory
title: 판매자 웹 서비스 API 인벤토리
type: web-api-integration-catalog
status: draft
tags: [web, seller, api, service, openapi, code]
source: local
created: 2026-07-13
updated: 2026-07-16
---

# 판매자 웹 서비스 API 인벤토리

## 확인 기준

- `service` checkout: 2026-07-16 HEAD `1bb90b3`
- `archive`: 2026-07-16 HEAD `7d2e7a5`
- `gitops`: 2026-07-16 HEAD `8e14539` + 기존 Auth values 작업 트리 변경. User·Coupon의 DropMong 표기 선언, ticketing/MediKong 표기의 동명 서비스와 미선언 서비스 확인. 라이브 동기화 미확인
- 상태 표기 순서: API 구현 / seller 사용 / Ingress / 프론트

## 현행 Seller BFF 코드 식별자

아래 진입점은 현재 코드에 있지만 목표 계약이 아니다.

| 구분 | Method / Path | 현재 함수 | 현재 동작 | 목표 처리 |
| --- | --- | --- | --- | --- |
| seller context | `GET /api/web/seller/context` | `toSellerContext` | 개발 seller session·permission 반환 | 실제 Auth와 membership 계약 뒤 제거 |
| onboarding | `POST /api/web/seller/onboarding` | Route Handler 분기 | active seller fixture session으로 전환 | 소유 서비스 결정 전 구현 금지 |
| browser Command | `POST /api/web/seller/{commandPath}` | `executeSellerCommand` | 웹 보안 검사 뒤 범용 client 호출 | resource별 canonical API로 교체 |
| page 조회 | `readSellerPage(kind)` | `getSellerPageFixture` 또는 `GET /web/seller/pages/{kind}` | 개발 fixture 또는 가상 downstream | PAGE별 실제 소유 서비스 계약으로 교체 |
| Command 대상 | `sendSellerCommand` | `POST /web/seller/commands/{commandPath}` | 개발 완료 fixture 또는 가상 downstream | 제거 |

`SELLER_CONTEXT_INTERNAL_BASE_URL`은 설정 검사만 있고 실제 호출 client가 없다. `SELLER_MANAGEMENT_INTERNAL_BASE_URL`은 단일 가상 page·command 서비스 placeholder다. 둘 다 목표 구성에 사용하지 않는다.

## 서비스별 API 상태

### Auth Service

| API | Method / Path | API 구현 | seller 사용 | Ingress | 프론트 |
| --- | --- | --- | --- | --- | --- |
| `API.A.300-16 getAuthContext` | `GET /api/v1/auth/context` | route·OpenAPI·HTTP E2E 구현 | 인증 subject 확인 보조만 | HEAD의 ticketing/MediKong 표기 `/auth`는 현재 작업 트리에서 비활성, canonical 경로·DropMong 동일성·실제 배포 미확인 | 개발 cookie, 실제 Auth 미연결 |
| `API.A.300-17 reauthenticateEmail` | `POST /api/v1/auth/reauthentications/email` | `link_identity`, `replace_phone` purpose 구현·E2E. seller purpose 없음 | 현재 seller 사용 불가 | HEAD의 ticketing/MediKong 표기 `/auth`는 현재 작업 트리에서 비활성, canonical 경로·DropMong 동일성·실제 배포 미확인 | 개발 fixture 재인증만 사용 |
| `API.A.300-29 resumeAuthenticatedAction` | `POST /api/v1/auth/intents/{intentId}/action-resume` | `purchase` action만 구현·E2E. seller variant 없음 | 현재 seller 사용 불가 | HEAD의 ticketing/MediKong 표기 `/auth`는 현재 작업 트리에서 비활성, canonical 경로·DropMong 동일성·실제 배포 미확인 | 미연결 |

`GET /api/v1/auth/context`의 현재 service 구현은 mobile credential 경계를 사용한다. 브라우저 opaque session 계약과 `dropmong-web` 연결은 추가 정합화가 필요하다. Auth는 seller membership·role·permission을 소유하지 않는다.

### Catalog Service

| API | Method / Path | API 구현 | seller 사용 | Ingress | 프론트 |
| --- | --- | --- | --- | --- | --- |
| 공개 Drop 목록 | `GET /drops` | 구현 | 불가. seller 소유·초안·검수 없음 | 선언 없음 | seller 미연결 |
| 공개 Drop 상세 | `GET /drops/{dropId}` | 구현 | 공개 결과 대조 외 seller 원장으로 사용 불가 | 선언 없음 | seller 미연결 |
| `API.A.200-09~18` | 논리 seller 상품·제안·검수·Drop | 없음 | 소유 후보 | 미연결 | fixture |

buyer API를 seller API처럼 재사용하지 않는다. 다른 seller 리소스와 미존재 리소스를 같은 `404`로 처리할 seller 소유권 계약도 아직 없다.

### Coupon Service

| API | Method / Path | API 구현 | seller 사용 | Ingress | 프론트 |
| --- | --- | --- | --- | --- | --- |
| `API.A.19-10` | `POST /api/v1/internal/coupon-campaigns` | 구현 | seller 공개 adapter·scope 필요 | DropMong `/coupons` 선언과 경로 불일치, seller 공개 없음 | fixture |
| `API.A.19-11` | `PUT /api/v1/internal/coupon-campaigns/{campaignId}/issuance-policy` | 구현 | seller scope 필요 | DropMong `/coupons` 선언과 경로 불일치, seller 공개 없음 | fixture |
| `API.A.19-12` | `POST /api/v1/internal/coupon-campaigns/{campaignId}/reviews` | 구현 | 판매자 직접 호출 대상 아님 | DropMong `/coupons` 선언과 경로 불일치, seller 공개 없음 | fixture |
| `API.A.19-13` | `POST /api/v1/internal/coupon-campaigns/{campaignId}/policy-versions` | 구현 | seller version 계약 필요 | DropMong `/coupons` 선언과 경로 불일치, seller 공개 없음 | fixture |
| `API.A.19-16` | `GET /api/v1/internal/coupon-campaigns/{campaignId}/performance` | 기본 집계 구현 | seller 집계·scope 보강 필요 | DropMong `/coupons` 선언과 경로 불일치, seller 공개 없음 | fixture |
| `API.A.19-25` | `GET /api/v1/internal/coupon-cost-attributions` | 구현 | seller 정산 원장 아님 | DropMong `/coupons` 선언과 경로 불일치, seller 공개 없음 | fixture |

seller별 캠페인 목록과 제휴 제안 조회·응답 API는 없다. `PAGE.A.206` 전체를 현재 internal API만으로 제공할 수 없다.

### Order Service

| API | Method / Path | API 구현 | seller 사용 | Ingress | 프론트 |
| --- | --- | --- | --- | --- | --- |
| buyer 주문 생성 | `POST /orders` | 구현 | 불가 | 선언 없음 | seller 미연결 |
| buyer 주문 단건 | `GET /orders/{orderId}` | 구현 | 불가. buyer 소유권만 검사 | 선언 없음 | seller 미연결 |
| `API.A.200-19~22` | seller 주문·export | 없음 | 소유 후보 | 미연결 | fixture |

seller 주문 목록, 개인정보 마스킹, 현재 membership, export 감사·수명주기 계약을 별도로 구현해야 한다.

### Payment Service

| API | Method / Path | API 구현 | seller 사용 | Ingress | 프론트 |
| --- | --- | --- | --- | --- | --- |
| 개발 결제 승인·실패 | `POST /payments/mock-approvals`, `/payments/mock-failures` | 구현 | 불가 | ticketing/MediKong 표기의 동명 `/payments` 선언, DropMong 동일성 미확인 | 미연결 |
| 구매자 결제 단건 | `GET /payments/{paymentId}` | 구현 | 불가 | ticketing/MediKong 표기의 동명 `/payments` 선언, DropMong 동일성 미확인 | 미연결 |

seller 브라우저와 웹 서버가 Payment를 직접 조회해 매출·실패율을 계산하지 않는다. 소유자가 결정된 집계 Read Model이 versioned Payment Event를 사용한다.

### Interest Service

| API | Method / Path | API 구현 | seller 사용 | Ingress | 프론트 |
| --- | --- | --- | --- | --- | --- |
| buyer 관심 | `PUT/DELETE /v1/users/me/interests/{dropId}`, `GET /v1/users/me/interests` | 구현 | 불가 | 선언 없음 | seller 미연결 |
| buyer 랭킹 | `GET /v1/rankings/drops/upcoming`, `/trending` | 구현 | seller 업무 API 아님 | 선언 없음 | seller 미연결 |
| operator 통계 | `GET /v1/operator/drops/{dropId}/interest-stats` | 구현 | operator 전용 | 선언 없음 | seller 미연결 |

현재 서비스 목록에는 포함되지만 seller dashboard·analytics 원장으로 직접 재사용하지 않는다.

### Notification Service

| API | Method / Path | API 구현 | seller 사용 | Ingress | 프론트 |
| --- | --- | --- | --- | --- | --- |
| buyer 알림 | `GET /notifications` | 구현 | 불가. seller principal·분류 없음 | ticketing/MediKong 표기의 동명 `/notifications` 선언, DropMong 동일성 미확인 | seller 미연결 |

seller 알림이나 Seller Issue 갱신을 제공하려면 seller principal과 알림 분류 계약이 필요하다.

### User Service

| API | Method / Path | API 구현 | seller 사용 | Ingress | 프론트 |
| --- | --- | --- | --- | --- | --- |
| 사용자 생성 | `POST /api/v1/users` | 구현 | seller account 생성 계약 아님 | DropMong exact route 선언, 실제 배포 미확인 | seller 미연결 |
| 본인 프로필 | `GET/PATCH /api/v1/users/me/profile` | 구현 | seller StoreProfile·membership 아님 | DropMong `/api/v1/users/me` Prefix 선언, 실제 배포 미확인 | seller 미연결 |
| 계정 상태 | `POST /api/v1/operator/users/{userId}/status-transitions`; `GET /api/v1/operator/users/{userId}/status` | 구현 | operator 사용자 계정 계약 | 해당 Ingress route 없음 | seller 미연결 |

User의 사용자 프로필과 계정 상태를 SellerAccount, StoreProfile, SellerMembership 원장으로 확장하지 않는다.

## 논리 판매자 계약 배치

| 필요한 경계 | 논리 API | 목표 소유 서비스 | API 구현 | seller·배포·프론트 상태 |
| --- | --- | --- | --- | --- |
| Seller Access·Management | `API.A.200-01~08` | 미확정 | 설계 초안 | seller 사용 불가 / Ingress 미연결 / 프론트 fixture |
| Seller Proposal | `API.A.200-09~16` | `catalog-service` 후보 | 없음 | seller 사용 불가 / Ingress 미연결 / 프론트 fixture |
| Seller Dashboard·Analytics | `API.A.200-17`, `23~24` | 미확정 | 없음 | seller 사용 불가 / Ingress 미연결 / 프론트 fixture |
| Seller Drop | `API.A.200-18` | `catalog-service` 후보 | 없음 | seller 사용 불가 / Ingress 미연결 / 프론트 fixture |
| Seller Order Export | `API.A.200-19~22` | `order-service` 후보 | 없음 | seller 사용 불가 / Ingress 미연결 / 프론트 fixture |
| Seller Issue | `API.A.200-25~28` | 미확정 | 없음 | seller 사용 불가 / Ingress 미연결 / 프론트 fixture |
| Seller Partnership | 제휴 제안 조회·응답 | `coupon-service` 검토 | 없음 | seller 사용 불가 / Ingress 미연결 / 프론트 fixture |

상세 method/path와 schema는 [operation catalog](../../../50-service-design/A_200_seller/A_200_40-api/operation-catalog.md), 외부 Event gap은 [Event 계약](../../../50-service-design/A_200_seller/A_200_40-api/event-contracts.md)을 따른다. 이 초안은 실제 소유 서비스 OpenAPI가 아니다.
