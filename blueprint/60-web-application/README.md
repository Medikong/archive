---
id: web-application-index
title: 웹 애플리케이션 설계 인덱스
type: web-application-index
status: active
tags: [web-application, frontend, nextjs, app-router, responsive]
source: local
created: 2026-07-10
updated: 2026-07-13
---

# 웹 애플리케이션 설계 인덱스

## 역할

이 폴더는 사이트맵과 UI 근거를 실제 웹 애플리케이션 구조로 옮길 때 필요한 구현 결정을 기록한다. 어떤 URL에 어떤 `PAGE`와 `UI`를 배치할지, Server Component와 Client Component를 어디에서 나눌지, 화면 상태와 서버 데이터를 어디에서 소유할지를 다룬다.

초기 기준은 구매자, 판매자, 플랫폼 운영자 화면을 하나의 반응형 Next.js App Router 애플리케이션으로 제공하는 것이다. 하나의 애플리케이션으로 시작하더라도 액터별 route group, layout, 권한, 컴포넌트 책임은 섞지 않는다.

## 읽기 순서

1. [사이트맵](../10-sitemap/README.md)에서 페이지 목적, URL, 페이지 사이 이동을 확인한다.
2. [UI](../20-ui/README.md)에서 화면 구성, 상태 표현, 에셋을 확인한다.
3. [WEB.A.01](WEB_A_01_frontend_architecture.md)에서 애플리케이션 구조와 렌더링 경계를 확인한다.
4. [WEB.A.02](WEB_A_02_state_data_strategy.md)에서 상태 소유권, 조회, 캐시, 재시도 정책을 확인한다.
5. [BFF.A.01](BFF_A_01_web_bff_module.md)에서 화면 DTO, 웹 세션, CSRF와 내부 API 호출 경계를 확인한다.
6. 액터별 구현을 시작할 때 [판매자 웹 애플리케이션 설계](A_03_seller/README.md)처럼 해당 클라이언트 상세 설계를 확인한다. 판매자 웹은 BFF를 사용하지 않고 Kubernetes Ingress를 통해 실제 소유 서비스를 호출한다.
7. [서비스 상세 설계](../50-service-design/README.md)와 [처리 시퀀스](../80-sequence/README.md)에서 API 계약과 여러 Context의 처리 순서를 확인한다.
8. [WEB.A.03](WEB_A_03_deployment_observability_test.md)에서 배포, 관측성, 카나리·롤백과 공통 테스트 기준을 확인한다.

## 기준 문서

| 판단 대상 | 기준 문서 | 이 폴더에서 다루는 내용 |
| --- | --- | --- |
| 사용자 문제, 제약, 수용 기준 | [00-requirements](../00-requirements/README.md) | 요구사항을 어떤 웹 경계에서 확인할지 연결한다. |
| 페이지 목적, URL, 이동 | [10-sitemap](../10-sitemap/README.md) | route group과 `page.tsx` 배치를 결정한다. |
| 화면 구성, 시각 상태, 에셋 | [20-ui](../20-ui/README.md) | layout과 컴포넌트가 사용할 UI 근거를 연결한다. |
| 사용자 목표와 예외 | [30-uc](../30-uc/README.md) | 제출, 구매, 재시도 같은 사용자 행동 경계를 확인한다. |
| 업무 규칙, 데이터, API | [50-service-design](../50-service-design/README.md) | 웹이 소비할 계약을 참조하며 규칙을 다시 정의하지 않는다. |
| 화면과 API의 처리 순서 | [80-sequence](../80-sequence/README.md) | 로딩, 실패, 완료 시점의 사용자 피드백을 연결한다. |
| route, 렌더링, 상태, 웹 캐시 | `60-web-application` | 웹 애플리케이션에 한정된 구현 결정을 소유한다. |
| 화면 DTO, 웹 세션, CSRF, 호출 조정 | [BFF.A.01](BFF_A_01_web_bff_module.md) | 같은 Next.js 배포 단위 안의 서버 모듈 경계를 소유한다. |
| 도메인 원장과 불변조건 | 각 업무 Context의 서비스 상세 설계 | BFF나 브라우저가 복제하지 않는다. |

`PAGE`는 페이지의 목적과 이동을, `UI`는 화면 근거를, 서비스 상세 설계는 업무 계약을 소유한다. `WEB` 문서는 이 내용을 복사해 새 기준 문서로 만들지 않는다. 대신 구현 선택과 원천 문서 링크만 기록한다.

## 문서

| Web ID | 문서 | 역할 | 상태 |
| --- | --- | --- | --- |
| `WEB.A.01` | [프론트엔드 애플리케이션 구조](WEB_A_01_frontend_architecture.md) | 단일 App Router 앱, route group, layout, 컴포넌트 경계, 상태 화면, 앱 분리 기준 | draft |
| `WEB.A.02` | [상태와 데이터 전략](WEB_A_02_state_data_strategy.md) | 서버 상태, URL 상태, 폼 상태, 로컬 UI 상태, 조회 캐시와 재시도 기준 | draft |
| `BFF.A.01` | [웹 BFF 애플리케이션 모듈](BFF_A_01_web_bff_module.md) | 화면 DTO, 웹 세션·CSRF, 내부 요청 변환과 제한된 호출 조정 | draft |
| `WEB.A.03` | [배포·관측성·테스트 설계](WEB_A_03_deployment_observability_test.md) | image, Kubernetes, 관측성, SLO, 카나리·롤백, Playwright 검증 | draft |
| `WEB.A.200` | [판매자 웹 포털](A_03_seller/WEB_A_200_seller_portal.md) | `PAGE.A.200~211` route, 셸, 권한, 상태, 반응형, 접근성, 검증 기준 | draft |
| `BFF.A.200` | [현행 판매자 BFF 코드 기록](A_03_seller/BFF_A_200_seller_portal_profile.md) | 현재 seller Route Handler·fixture와 제거 조건. 목표 구조에서는 사용하지 않음 | retired-target |

## 클라이언트별 상세 설계

| 요구사항 기준 | 설계 | 배포 기준 |
| --- | --- | --- |
| `REQ.A.03` 판매자 | [A_03_seller](A_03_seller/README.md) | 구매자 웹과 같은 Next.js 애플리케이션·이미지를 사용하되 Seller BFF 없이 브라우저 → Kubernetes Ingress → 실제 MSA 서비스로 연결한다. |

## 템플릿

- [웹 애플리케이션 설계 템플릿](.template/WEB_A_XX_web_application.md)
- [웹 BFF 애플리케이션 모듈 템플릿](.template/BFF_A_XX_bff_module.md)

## 식별자와 파일명

- 웹 애플리케이션 설계는 `WEB` prefix를 사용한다.
- 문서 식별자는 `WEB.A.01`, 파일명은 `WEB_A_01_frontend_architecture.md`처럼 적는다.
- 같은 배포 단위의 웹 BFF 모듈은 `BFF` prefix를 사용하고 도메인 서비스 식별자로 사용하지 않는다.
- 특정 페이지 구현만 다루더라도 새 `PAGE` 식별자를 만들지 않고 기존 `PAGE`와 `UI`를 링크한다.
- 공통 정책이 바뀌면 `WEB` 문서를 갱신하고, 페이지 목적이나 화면 근거가 바뀌면 원천 `PAGE` 또는 `UI` 문서를 먼저 갱신한다.

## 포함 범위

- Next.js App Router의 route group, segment, layout, loading, error, not-found 배치
- Server Component와 Client Component의 책임 분리
- 페이지별 데이터 조회와 mutation 진입점
- 서버 상태, URL 상태, 폼 상태, 로컬 UI 상태의 소유권
- 웹 캐시, 재시도, polling, 무효화, 최신 시각 표시 기준
- 화면 DTO, 웹 세션, CSRF와 내부 API 호출 조정 기준
- 반응형 화면과 액터별 셸의 구현 경계
- 단일 애플리케이션을 별도 앱으로 나누는 판단 기준

## 제외 범위

- 페이지의 존재 이유와 이동 규칙 재정의
- 이미지, 색상, 타이포그래피와 컴포넌트 시안 제작
- 도메인 규칙, API 요청/응답, 데이터베이스 스키마 재정의
- BFF 전용 업무 원장, Aggregate, 장기 실행 작업 추가
- 클라이언트 상세 문서에서 Kubernetes, Ingress, 배포 전략과 관측성 스택의 공통 기준 재정의. 공통 기준은 `WEB.A.03`이 소유한다.
- 클라이언트 상세 문서에서 Playwright 실행 기반과 공통 브라우저 매트릭스 재정의. 상세 문서는 해당 클라이언트의 검증 시나리오만 추가한다.

## 작성 원칙

- 구현 편의를 이유로 `PAGE`, `UI`, API 계약의 의미를 바꾸지 않는다.
- Server Component를 기본값으로 두고, 상호작용이나 브라우저 API가 필요한 가장 작은 컴포넌트에만 `'use client'`를 둔다.
- 주문, 결제, 재고, 배송, 권한처럼 서버가 판정하는 상태를 브라우저 전역 store의 권위 상태로 사용하지 않는다.
- 로딩, 오류, 빈 결과, 최신성 지연은 서로 다른 화면 상태로 설계한다.
- 구매자, 판매자, 플랫폼 운영자는 한 코드베이스를 쓰더라도 route group과 layout에서 책임을 분리한다.
- 앱 분리는 팀 수가 아니라 배포, 보안, 장애 영향, 성능 지표가 실제로 달라졌다는 근거로 결정한다.

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.01](../00-requirements/REQ_A_01_limited_drop_commerce.md), [REQ.A.03](../00-requirements/REQ_A_03_seller.md), [REQ.A.04](../00-requirements/REQ_A_04_platform_operator_admin.md), [REQ.A.05](../00-requirements/REQ_A_05_auth_member.md), [REQ.A.08](../00-requirements/REQ_A_08_web_application.md) | 페이지 참조: [사이트맵 인덱스](../10-sitemap/README.md) | UI 참조: [UI 인덱스](../20-ui/README.md) | UC 참조: [유스케이스 인덱스](../30-uc/INDEX.md) | 판매자 웹 참조: [WEB.A.200](A_03_seller/WEB_A_200_seller_portal.md), [현행 BFF 기록](A_03_seller/BFF_A_200_seller_portal_profile.md) | 서비스 참조: [서비스 상세 설계](../50-service-design/README.md) | 시나리오 참조: [처리 시퀀스](../80-sequence/README.md)
