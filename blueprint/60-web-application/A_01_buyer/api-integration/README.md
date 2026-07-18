---
id: buyer-web-api-integration-index
title: 구매자 웹 API 연동 인벤토리
type: web-api-integration-index
status: draft
tags: [web, buyer, bff, api, integration, traceability]
source: local
created: 2026-07-16
updated: 2026-07-16
page_ids: [PAGE.A.01, PAGE.A.02, PAGE.A.06, PAGE.A.09, PAGE.A.10, PAGE.A.11, PAGE.A.14, PAGE.A.15, PAGE.A.16, PAGE.A.17, PAGE.A.19, PAGE.A.22, PAGE.A.23]
---

# 구매자 웹 API 연동 인벤토리

## 역할

이 폴더는 구매자 PAGE와 현재 `dropmong-web` 함수·Route Handler, 실제 서비스 API, 로컬 실행 설정, GitOps·Ingress와 fixture 의존성을 대조한다.

## 확인 기준

| 원천 | ref | 확인 항목 |
| --- | --- | --- |
| `service` checkout | 2026-07-16 HEAD `1bb90b3` | 웹 route·함수·환경 변수·테스트와 서비스 API |
| `service/config/services.yml` | 같은 checkout | image build·게시 목록. 배포 완료 증거는 아님 |
| `archive` | 2026-07-16 HEAD `7d2e7a5` | buyer PAGE·UI와 목표 책임 |
| `gitops` | 2026-07-16 HEAD `8e14539` + 기존 Auth values 작업 트리 변경 | 서비스별 선언, Ingress 경로, DropMong 동일성과 Auth 변경 전후 |

## 문서

| 문서 | 내용 |
| --- | --- |
| [서비스 API 인벤토리](SERVICE_API_INVENTORY.md) | 서비스별 API 구현, 웹 코드, 로컬·CI·배포와 프론트 상태 |
| [페이지별 API 매트릭스](PAGE_API_MATRIX.md) | PAGE route, 현재 함수·Route Handler, 목표 소유자, fixture와 다음 작업 |

## 상태 축

| 축 | 값 | 의미 |
| --- | --- | --- |
| 서비스 API | 구현 / 일부 / 없음 / 소유 미확정 | 실제 서비스 코드·OpenAPI·테스트 존재 여부 |
| 웹 코드 | 지원 / 없음 | `dropmong-web` client·함수·Route Handler 존재 여부 |
| 프론트 | 실연결 / fixture / 미연결 | PAGE가 canonical API를 실제 사용하는지 여부 |
| 로컬 | 실서비스 / mock / 미설정 | `.env.local`과 실행 설정 |
| 검증 | 실연동 / mock만 / 없음 | CI·브라우저·Docker 검증 범위 |
| 배포 | 연결 / 미연결 / 미확인 | DropMong GitOps와 Ingress 근거 |

한 축이 완료됐다는 이유로 다른 축을 완료로 바꾸지 않는다.

## 설정 감사

| 환경 | `DEV_MOCK_MODE` | Catalog URL | Auth·Order·Payment URL | 판정 |
| --- | --- | --- | --- | --- |
| `.env.local` | `true` | 주석 | 없음 | 로컬 mock |
| GitHub Actions | `true` | 없음 | 없음 | mock 검증만 |
| Playwright | `true` | 없음 | 없음 | fixture buyer·seller 화면 |
| Docker smoke | `true` | 없음 | 없음 | 기동·health·ready·metrics·요청 완료 로그만 |
| GitOps | `dropmong-web` 선언 없음 | Catalog URL 없음 | Auth·Order·Payment URL 없음 | web 실연동 미연결, 서비스별 배포 선언은 원장에서 별도 판정 |

`/readyz`는 설정 parsing과 요청 수락 상태를 확인할 뿐 downstream 연결을 검증하지 않는다.

GitOps에는 DropMong 표기의 User·Coupon private-dev 선언이 있다. User Ingress는 `/api/v1/users`, `/api/v1/users/me`를 포함하지만 실제 동기화는 미확인이다. Coupon의 `/coupons` route는 현재 canonical `/api/v1/**` 경로와 일치하지 않는다. Auth·Payment·Notification의 ticketing/MediKong 표기 동명 선언은 현재 DropMong checkout과의 동일성을 확인하기 전까지 배포 근거로 사용하지 않는다. Auth HEAD의 `/auth` route는 기존 작업 트리 변경에서 비활성이다.

## 현재 우선순위

1. Catalog
2. Auth
3. Checkout 계약
4. Order·Payment 상태
5. User·Interest·Coupon·Notification 부가 Query

앞 단계의 소유권과 오류 계약이 확정되지 않은 상태에서 다음 단계를 fixture 성공으로 연결하지 않는다.

## Catalog 첫 연동 시작 조건

다음 작업자는 서비스 코드를 수정하기 전에 이 범위만 구현·검증 대상으로 잡는다.

1. `dropmong-web` 실행 환경에 `CATALOG_INTERNAL_BASE_URL`을 명시한다.
2. `/`의 `getHomePage`와 `/products/[productId]`의 `getProductDetailPage`가 실제 `GET /drops`, `GET /drops/{dropId}`를 사용하는지 확인한다.
3. `DEV_MOCK_MODE=true`여도 Catalog URL이 있으면 실제 응답을 사용한다는 현재 우선순위를 보존한다.
4. upstream 연결 실패·timeout·잘못된 schema가 fixture로 바뀌지 않고 typed 오류로 처리되는지 확인한다.
5. 별도 실제 연동 테스트를 추가하고 기존 mock CI·Docker smoke와 검증 이름을 구분한다.
6. DropMong 배포 환경을 만들 때 Catalog service DNS, 환경 변수와 Ingress 필요 여부를 GitOps에 명시한다.

현재 `.env.local`을 이 문서 작업에서 수정하지 않는다.

## 갱신 규칙

- 새 PAGE route가 생기면 같은 변경에서 [페이지별 API 매트릭스](PAGE_API_MATRIX.md)에 함수와 handler를 기록한다.
- 서비스 endpoint가 생겨도 웹 client가 없으면 `프론트 미연결`로 둔다.
- CI가 mock을 쓰면 실제 API 코드가 있더라도 `실연동 검증`으로 표시하지 않는다.
- GitOps 선언의 service identity, route 포함 범위와 실제 동기화를 확인하지 않은 채 image inventory나 동명 서비스를 배포 증거로 사용하지 않는다.
- Checkout·배송·포인트·결제수단의 소유자를 추정하지 않는다.
