---
id: SD.A.20040
title: Context 판매자 API 설계 인덱스
type: service-design-api-index
status: draft
tags: [service-design, seller, api, openapi, ingress]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
api_ids: [API.A.200-01, API.A.200-02, API.A.200-03, API.A.200-04, API.A.200-05, API.A.200-06, API.A.200-07, API.A.200-08, API.A.200-09, API.A.200-10, API.A.200-11, API.A.200-12, API.A.200-13, API.A.200-14, API.A.200-15, API.A.200-16, API.A.200-17, API.A.200-18, API.A.200-19, API.A.200-20, API.A.200-21, API.A.200-22, API.A.200-23, API.A.200-24, API.A.200-25, API.A.200-26, API.A.200-27, API.A.200-28]
---

# Context 판매자 API 설계 인덱스

## 역할과 상태

`API.A.200-01~28`은 하나의 `seller-service`가 제공하는 API가 아니다. `CMD.A.200-01~17`과 `RM.A.200-01~10`을 빠짐없이 식별하기 위한 논리 operation catalog다. 브라우저는 목표 구조에서 Seller BFF를 거치지 않고 Kubernetes Ingress를 통해 실제 소유 서비스의 공개 seller API를 호출한다.

현재 분리형 OpenAPI는 method, schema, 오류와 동시성 조건을 검토하기 위한 `allocation-pending` 초안이다. `/api/v1/internal` path, `WorkloadBearerAuth`와 `X-Seller-Scope`는 기존 BFF 전제를 반영하므로 배포 계약으로 사용할 수 없다. 각 operation의 소유 서비스와 Ingress-facing 인증 계약이 확정되면 해당 서비스 OpenAPI로 옮기고 이 초안과의 중복을 제거한다.

| 원장 | 내용 |
| --- | --- |
| [Operation catalog](operation-catalog.md) | `API.A.200-01~28` method/path 초안, 권한, request/response, 오류, 멱등·version, 목표 소유 서비스 |
| [Event 계약](event-contracts.md) | `EVT.A.200-01~17`과 외부 수신 Event의 envelope·보장·계약 대기 상태 |
| [PAGE/API 추적성](page-api-traceability.md) | `PAGE.A.200~211` → 현행 코드 식별자 → 논리 API → CMD/RM → 실제 MSA·상태 |
| [OpenAPI 초안](openapi/openapi.yaml) | 소유 서비스 배치 전 wire 검토 자료. 현재 배포 불가 |

## 실제 서비스 배치

| 범위 | API | 목표 배포 단위 | 호출 주체 | 상태 |
| --- | --- | --- | --- | --- |
| Access·Management | `API.A.200-01~08` | 미확정 | 브라우저 → Ingress → 소유 서비스 | 배포 불가 |
| Proposal | `API.A.200-09~16` | `catalog-service` 후보 | 브라우저 또는 검수·handoff worker | seller 계약 미구현 |
| Operations Query | `API.A.200-17`, `API.A.200-23~24` | 미확정 | 브라우저 → Ingress → 소유 서비스 | 교차 서비스 조회 모델 소유자 필요 |
| Seller Drop | `API.A.200-18` | `catalog-service` 후보 | 브라우저 → Ingress → Catalog | seller 계약 미구현 |
| Seller Order·Export | `API.A.200-19~22` | `order-service` 후보 | 브라우저, export worker, expiry scheduler | seller 계약 미구현 |
| Seller Issue | `API.A.200-25~28` | 미확정 | 브라우저 또는 platform operations worker | 배포 불가 |

## Ingress-facing 계약 원칙

- 외부 진입은 Kubernetes Ingress가 소유하고, 업무 path와 응답은 실제 서비스 OpenAPI가 소유한다.
- Ingress는 외부에서 주입한 사용자·seller header를 제거하고 Auth가 검증한 사용자 context만 전달한다.
- 브라우저는 업무 서비스의 seller operation을 호출하되, 수신 서비스가 현재 membership, permission version과 resource seller ownership을 다시 검증한다.
- 인증 context 전달 형식과 seller membership 조회 소유자가 확정되기 전에는 보호 seller route를 공개하지 않는다.
- Query는 `Cache-Control: private, no-store`, `ETag`, `meta.asOf`와 section freshness를 사용한다.
- Command는 `Idempotency-Key`와 `If-Match` 또는 명시적 expected version을 사용한다.
- 다른 seller 소유와 존재하지 않는 대상은 동일한 `404` 응답으로 처리한다.
- 비동기 export 접수는 `202 Accepted`이며 `READY` 성공으로 표시하지 않는다.
- 브라우저, Ingress와 `dropmong-web`은 Order·Payment·Catalog·Coupon을 fan-out해 KPI를 계산하지 않는다.

## 검증

```bash
npx --yes @redocly/cli@2.38.0 lint \
  --config blueprint/50-service-design/A_200_seller/A_200_40-api/openapi/redocly.yaml \
  blueprint/50-service-design/A_200_seller/A_200_40-api/openapi/openapi.yaml

npx --yes @redocly/cli@2.38.0 bundle \
  blueprint/50-service-design/A_200_seller/A_200_40-api/openapi/openapi.yaml \
  --output /tmp/A_200_seller.openapi.bundle.yaml
```

bundle은 검증 생성물이며 저장소 원장으로 직접 수정하지 않는다.
