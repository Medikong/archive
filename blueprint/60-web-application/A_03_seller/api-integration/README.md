---
id: seller-web-api-integration-index
title: 판매자 웹 API 연동 인벤토리
type: web-api-integration-index
status: draft
tags: [web, seller, ingress, api, integration, traceability]
source: local
created: 2026-07-13
updated: 2026-07-16
page_ids: [PAGE.A.200, PAGE.A.201, PAGE.A.202, PAGE.A.203, PAGE.A.204, PAGE.A.205, PAGE.A.206, PAGE.A.207, PAGE.A.208, PAGE.A.209, PAGE.A.210, PAGE.A.211]
---

# 판매자 웹 API 연동 인벤토리

## 역할

이 폴더는 판매자 PAGE와 실제 MSA 계약 사이의 상태 원장이다. 현재 Seller BFF의 page kind와 command path는 코드 제거를 위한 식별자로만 기록하고 목표 API로 사용하지 않는다.

## 확인 기준

| 원천 | ref | 확인 결과 |
| --- | --- | --- |
| `service` checkout | 2026-07-16 HEAD `1bb90b3` | 현재 API·테스트와 `dropmong-web` 코드 |
| `service/config/services.yml` | 같은 checkout | auth, catalog, coupon, dropmong-web, interest, notification, order, payment, user 9개 image |
| `archive` | 2026-07-16 HEAD `7d2e7a5` | 목표 소유권과 `API.A.200-*` 논리 operation |
| `gitops` | 2026-07-16 HEAD `8e14539` + 기존 Auth values 작업 트리 변경 | User·Coupon의 DropMong 표기 선언, ticketing/MediKong 표기의 동명 서비스, 미선언 서비스와 Auth 변경 전후 확인 |

서비스 image inventory는 배포 증거가 아니다. GitOps 선언이 있어도 현재 DropMong checkout의 image 동일성, seller route 포함 범위와 실제 동기화는 별도로 확인한다. 라이브 클러스터는 이번 범위에 포함하지 않았다.

## 문서

| 문서 | 내용 |
| --- | --- |
| [서비스 API 인벤토리](SERVICE_API_INVENTORY.md) | 실제 endpoint, API 구현, seller 사용, Ingress, 프론트 상태 |
| [페이지별 API 매트릭스](PAGE_API_MATRIX.md) | 12개 PAGE route, 현재 함수·fixture와 목표 소유 서비스 |

## 상태 축

| 축 | 값 | 의미 |
| --- | --- | --- |
| API 구현 | 구현됨 / 일부 / 없음 | 소유 서비스 코드·OpenAPI·테스트의 operation 존재 여부 |
| seller 사용 | 가능 / 보조만 / 불가 / 소유 미확정 | seller scope·membership·소유권·마스킹·감사 충족 여부 |
| Ingress | 공개 / 미연결 / 미확인 | DropMong GitOps route와 신뢰 경계 여부 |
| 프론트 | 실연결 / fixture / 미연결 | canonical API를 실제 seller PAGE가 호출하는지 여부 |

`API 구현됨`이더라도 seller 사용, Ingress와 프론트가 준비되지 않으면 `연결됨`으로 쓰지 않는다.

## 현재 결론

- 목표 구조에서 Seller BFF를 사용하지 않는다. 브라우저 API는 Ingress를 거쳐 실제 소유 서비스로 전달한다.
- 현재 seller PAGE는 모두 `/api/web/seller/**`, `src/server/bff/seller/**` 또는 fixture에 의존하며 실제 서비스에는 연결되지 않았다.
- Auth에는 mobile context, `link_identity`·`replace_phone` 재인증과 `purchase` action resume가 구현됐다. seller purpose·action-resume variant는 없고 웹도 개발 cookie·fixture를 사용한다. Auth는 seller membership·permission 원장이 아니다.
- Catalog와 Order는 buyer API만 있어 seller 화면에서 사용할 수 없다.
- Coupon internal API 일부는 구현됐지만 seller 공개 계약, 목록, 제휴와 scope가 부족하다.
- Interest와 User는 현재 서비스 목록에 포함되지만 seller 업무 원장으로 재사용하지 않는다.
- Payment는 seller 브라우저가 직접 호출하지 않는다.
- User·Coupon에는 DropMong 표기의 일부 배포·Ingress 선언이 있지만 seller 전용 계약을 포함하지 않는다. Auth·Payment·Notification은 ticketing/MediKong 표기의 동명 선언과 현재 DropMong 코드의 동일성이 미확인이고, Catalog·Order·Interest·dropmong-web 선언은 찾지 못했다. Auth HEAD의 `/auth` route는 기존 작업 트리 변경에서 비활성이다.

## 연동 우선순위

1. DropMong GitOps와 Ingress의 인증 context, 외부 header 제거, CORS·CSRF와 감사 IP 계약을 정의한다.
2. seller membership·permission 원장의 실제 소유 서비스를 결정한다.
3. Catalog, Order, Coupon과 Notification의 seller operation을 각 소유 서비스 OpenAPI·테스트에 추가한다.
4. 통합 dashboard·analytics·settlement·Seller Issue의 소유 서비스 또는 제공 제외 범위를 결정한다.
5. 네 상태 축이 모두 준비된 PAGE부터 현행 Seller BFF와 fixture 의존을 제거한다.

## 갱신 규칙

- 새 `API.A.200-*`는 논리 operation을 먼저 정리한 뒤 실제 소유 서비스 계약에 배치한다.
- 코드 함수명, page kind와 `commandPath`는 canonical API ID로 다루지 않는다.
- seller scope가 없는 buyer API를 조합해 seller API처럼 사용하지 않는다.
- 브라우저, Server Component, Ingress와 `dropmong-web`은 여러 원천을 fan-out해 dashboard·analytics·settlement를 계산하지 않는다.
- API 코드, actor 적합성, GitOps와 프론트 변경 중 하나라도 바뀌면 두 원장을 같은 변경에서 갱신한다.
