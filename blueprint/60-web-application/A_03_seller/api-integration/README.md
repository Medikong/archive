---
id: seller-web-api-integration-index
title: 판매자 웹 API 연동 인벤토리
type: web-api-integration-index
status: draft
tags: [web, seller, ingress, api, integration, traceability]
source: local
created: 2026-07-13
updated: 2026-07-13
page_ids: [PAGE.A.200, PAGE.A.201, PAGE.A.202, PAGE.A.203, PAGE.A.204, PAGE.A.205, PAGE.A.206, PAGE.A.207, PAGE.A.208, PAGE.A.209, PAGE.A.210, PAGE.A.211]
---

# 판매자 웹 API 연동 인벤토리

## 역할

이 폴더는 판매자 브라우저가 Kubernetes Ingress를 통해 실제 MSA 서비스와 연결되는 상태를 대조한다. 현재 Seller BFF의 page kind와 command path는 코드 제거를 위한 식별자로만 기록하고 목표 API로 사용하지 않는다.

| 원천 | 확인 기준 | 상태 |
| --- | --- | --- |
| `service` 메인 `2070c20` | `dropmong-web`, Auth, Coupon, Catalog, Order, Payment, Notification 코드와 OpenAPI | 현재 메인 |
| `service/config/services.yml` | 실제 빌드·테스트 대상 서비스 목록 | Seller 서비스 없음 |
| Kubernetes GitOps | Kong Ingress class와 서비스별 route 패턴 | 앞단 경계 존재, seller route 미확정 |
| [SD.A.200](../../../50-service-design/A_200_seller/README.md) | 현재 MSA 배치, `CMD.A.200-01~17`, `RM.A.200-01~10` | 설계 원장 |
| [Seller API 초안](../../../50-service-design/A_200_seller/A_200_40-api/README.md) | `API.A.200-01~28` 논리 operation과 배치 상태 | 단일 런타임 계약 아님 |
| [현행 BFF 기록](../BFF_A_200_seller_portal_profile.md) | 현재 route, fixture, 제거 조건 | 목표에서 사용하지 않음 |

## 문서

| 문서 | 내용 |
| --- | --- |
| [서비스 API 인벤토리](SERVICE_API_INVENTORY.md) | 실제 구현 endpoint, seller 사용 가능 여부, 목표 소유 서비스 |
| [페이지별 API 매트릭스](PAGE_API_MATRIX.md) | 12개 PAGE의 Read Model, 현재 코드 식별자, 목표 MSA와 계약 부족 상태 |

## 상태 분류

| 상태 | 의미 |
| --- | --- |
| 연결 가능 | 메인 코드와 Ingress-facing 계약이 있고 seller 권한·범위를 충족함 |
| 일부 구현 | operation은 있으나 seller scope, 공개 path 또는 업무 변형이 부족함 |
| 설계 초안 | 논리 API와 schema가 있으나 소유 서비스 OpenAPI에 없음 |
| 소유 미확정 | 현재 MSA 어디에도 업무 원장이나 조회 모델을 배정하지 않음 |
| 사용 불가 | buyer·mock·operator 전용이어서 seller 화면에 사용할 수 없음 |

## 현재 결론

- 별도 `seller-service`를 추가하지 않는다. 현재 서비스별로 책임을 배정하고, 배정할 수 없는 항목은 소유 미확정으로 남긴다.
- 목표 구조에서 Seller BFF를 사용하지 않는다. 브라우저 API는 Ingress를 거쳐 실제 소유 서비스로 전달된다.
- Auth는 인증과 목적 한정 재인증만 담당한다. seller membership과 업무 permission 원장은 아직 소유 서비스가 없다.
- Catalog는 seller 상품·Drop·제안의 배정 후보지만 현재 공개 buyer Drop 조회만 구현되어 있다.
- Order는 seller 주문·export 배정 후보지만 현재 구매자 주문 계약만 구현되어 있다.
- Coupon의 `API.A.19-10~13/16/25`는 일부 구현되어 있으나 seller 공개 계약, 목록, 위험 승인·제휴 보강이 필요하다.
- Payment는 브라우저가 직접 호출하지 않는다. seller 귀속 결제 Event가 필요하다.
- dashboard·analytics·settlement·issue는 소유 서비스가 없으므로 운영 연결할 수 없다.

## 연동 우선순위

1. Ingress의 인증 context 전달, 외부 header 제거, CORS·CSRF와 감사 IP 계약을 확정한다.
2. seller membership·permission 원장의 실제 소유 서비스를 결정한다.
3. Catalog, Order, Coupon, Notification의 seller operation을 각 소유 서비스 OpenAPI에 추가한다.
4. 통합 조회와 Seller Issue의 소유 서비스 또는 제공 제외 범위를 결정한다.
5. 실제 API가 준비된 PAGE부터 `/api/web/seller/**`, page fixture와 범용 command path 의존을 제거한다.

## 갱신 규칙

- 새 `API.A.200-*`는 논리 operation catalog를 먼저 정리한 뒤 실제 소유 서비스 계약에 배치한다.
- 코드 함수명, page kind와 `commandPath`는 canonical API ID로 다루지 않는다.
- seller scope가 없는 buyer API를 조합해 seller API처럼 사용하지 않는다.
- 브라우저, Ingress와 `dropmong-web`은 Order·Payment·Catalog·Coupon을 fan-out해 대시보드·분석을 계산하지 않는다.
- 구현됨, 일부 구현, 설계 초안, 소유 미확정과 외부 계약 대기를 구분한다.
