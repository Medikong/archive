---
id: service-design-index
title: 서비스 상세 설계 인덱스
type: service-design-index
status: active
tags: [service-design, bounded-context, domain-model, persistence, service, api, sequence]
source: local
created: 2026-07-06
updated: 2026-07-16
---

# 서비스 상세 설계 인덱스

## 역할

이 폴더는 이벤트스토밍에서 분해한 바운디드 컨텍스트별 상세 설계를 모은다. MVP 전체 이벤트스토밍 문서 하나를 그대로 서비스 설계로 옮기지 않고, `Context 인증`, `Context 쿠폰`처럼 책임 경계가 나뉜 단위마다 도메인 모델, 영속성, 서비스, API, 처리 시퀀스를 설계한다.

## 폴더 구조

```text
50-service-design/
  <context>/
    README.md
    <context_id>_10-domain-model/
    <context_id>_20-persistence/
    <context_id>_30-service/
    <context_id>_40-api/
    <context_id>_50-sequence/
    <context_id>_60-deployment/
```

## 하위 설계 번호

| Suffix | 이름 | 역할 |
| --- | --- | --- |
| `10` | domain-model | Aggregate, Entity, Value Object, Domain Event, Policy, Business Rule 설계 |
| `20` | persistence | Repository, Database Schema, Index, Migration, 저장 경계 설계 |
| `30` | service | Application Service, Domain Service, Command Handler, Transaction Boundary 설계 |
| `40` | api | REST API, 이벤트 형식, 요청/응답, 오류 형식 설계 |
| `50` | sequence | 서비스 경계를 포함한 주요 처리 순서와 실패 처리 설계 |
| `60` | deployment | 배포 전략, migration, 가용성, 롤백과 운영 검증 설계 |

## 작성 단위

- `40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md`처럼 전체 MVP를 다루는 문서는 서비스 설계 폴더 하나로 옮기지 않는다.
- 전체 이벤트스토밍에서 식별한 바운디드 컨텍스트를 다시 분해해 `A_300_auth/`, `A_19_coupon/`, `<context_id>_<context_name>/` 같은 단위로 설계한다.
- 설계 분야 하나가 커지면 해당 번호 폴더 안에서 파일을 여러 개로 나눈다.
- 도메인 모델 폴더 안에서 Aggregate, Entity, Value Object 문서를 반드시 분리하지 않는다. Aggregate 단위 문서 안에 Entity와 Value Object를 함께 정의할 수도 있고, 작으면 단일 도메인 모델 문서 하나로 끝낼 수도 있다.
- 같은 컨텍스트의 설계 문서는 서로 가까운 상대 경로로 연결하고, 요구사항/UC/BC/시나리오는 원천 문서 링크로 연결한다.

## 공통 API 오류 응답

모든 Context의 공개 HTTP API는 다음 네 필드로 구성된 `ErrorResponse`를 공통 오류 본문으로 사용한다.

```json
{
  "code": "AUTH_SIGNIN_FAILED",
  "status": 401,
  "message": "입력한 정보를 확인한 뒤 다시 시도해주세요.",
  "requestId": "30d9fa85-0a18-4263-98b6-231dca5a6fb8"
}
```

- `code`는 클라이언트가 분기할 수 있는 안정적인 공개 코드다. 서비스별 namespace를 사용할 수 있도록 대문자 underscore와 소문자 dot 표기를 모두 허용한다.
- `status`는 실제 HTTP 응답 상태와 같아야 한다.
- `message`는 사용자에게 공개해도 안전한 설명이며 개인정보, credential, 내부 오류 문자열과 stack trace를 포함하지 않는다.
- `requestId`는 응답의 `X-Request-Id`와 같아야 한다.
- 임의의 `details` 객체는 두지 않는다. 재시도 시각과 rate limit 정보는 `Retry-After` 같은 표준 HTTP header로 제공한다.
- 공통 필드가 더 필요하면 특정 서비스가 독자적으로 추가하지 않고 이 구조의 version을 올린다.

[Context 인증 ErrorResponse schema](A_300_auth/A_300_40-api/openapi/components/schemas.yaml)는 이 공통 구조를 먼저 적용한 기계 판독 가능한 명세다. 아직 다른 형식을 사용하는 Context는 해당 API를 변경할 때 이 구조로 이전한다.

## 식별자 규칙

서비스 상세 설계는 `SD` prefix를 사용한다. 컨텍스트 루트 문서는 기존 업무 번호를 그대로 쓰고, 하위 설계 영역은 업무 번호 뒤에 `10`, `20`, `30`, `40`, `50`, `60`을 붙인다.

| 대상 | 예시 | 설명 |
| --- | --- | --- |
| Context 인증 루트 | `SD.A.300` | `A_300_auth/README.md` |
| Context 인증 도메인 모델 | `SD.A.30010` | `A_300_auth/A_300_10-domain-model/SD_A_30010_auth_domain_model.md` |
| Context 인증 영속성 | `SD.A.30020` | `A_300_auth/A_300_20-persistence/README.md` |
| Context 인증 서비스 | `SD.A.30030` | `A_300_auth/A_300_30-service/README.md` |
| Context 인증 API | `SD.A.30040` | `A_300_auth/A_300_40-api/README.md` |
| Context 인증 처리 시퀀스 | `SCN.A.300` | `A_300_auth/A_300_50-sequence/README.md` |
| Context 사용자 배포 | `SD.A.0160` | `A_01_user/A_01_60-deployment/README.md` |

폴더명은 컨텍스트 식별자와 하위 설계 번호를 함께 드러낸다. 예를 들어 Context 인증 도메인 모델 영역은 `A_300_10-domain-model/`이고, 문서 식별자는 `SD.A.30010`이다.

## 현재 컨텍스트

| 컨텍스트 폴더 | 바운디드 컨텍스트 | 상태 | 주요 근거 |
| --- | --- | --- | --- |
| [A_01_user](A_01_user/README.md) | Context 사용자 | draft | [BC.A.01](../40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md), [REQ.A.05](../00-requirements/REQ_A_05_auth_member.md), [도메인 모델](A_01_user/A_01_10-domain-model/README.md), [영속성](A_01_user/A_01_20-persistence/README.md), [서비스](A_01_user/A_01_30-service/README.md), [API](A_01_user/A_01_40-api/README.md), [배포](A_01_user/A_01_60-deployment/README.md) |
| [A_300_auth](A_300_auth/README.md) | Context 인증 | draft | [BC.A.300](../40-event-storming-bounded-context/BC_A_300_auth_member.md), [도메인 모델](A_300_auth/A_300_10-domain-model/SD_A_30010_auth_domain_model.md), [영속성](A_300_auth/A_300_20-persistence/README.md), [서비스](A_300_auth/A_300_30-service/README.md), [API](A_300_auth/A_300_40-api/README.md), [처리 시퀀스](A_300_auth/A_300_50-sequence/README.md) |
| [A_19_coupon](A_19_coupon/README.md) | Context 쿠폰 | draft | [BC.A.19](../40-event-storming-bounded-context/BC_A_19_coupon.md), [REQ.A.02](../00-requirements/REQ_A_02_coupon_benefit.md), [도메인 모델](A_19_coupon/A_19_10-domain-model/README.md), [영속성](A_19_coupon/A_19_20-persistence/README.md), [서비스](A_19_coupon/A_19_30-service/README.md), [API](A_19_coupon/A_19_40-api/README.md) |
| [A_07_interest_ranking](A_07_interest_ranking/README.md) | Context 찜 / Context 랭킹 집계 | draft | [BC.A.07](../40-event-storming-bounded-context/BC_A_07_interest_ranking.md), [REQ.A.07](../00-requirements/REQ_A_07_interest_ranking.md), [도메인 모델](A_07_interest_ranking/A_07_10-domain-model/SD_A_0710_interest_domain_model.md), [영속성](A_07_interest_ranking/A_07_20-persistence/persistence-design.md), [서비스](A_07_interest_ranking/A_07_30-service/service-design.md), [API](A_07_interest_ranking/A_07_40-api/README.md) |
| [A_200_seller](A_200_seller/README.md) | Context 판매자 | draft | [REQ.A.03](../00-requirements/REQ_A_03_seller.md), [BC.A.200](../40-event-storming-bounded-context/BC_A_200_seller.md), [도메인 모델](A_200_seller/A_200_10-domain-model/README.md), [영속성](A_200_seller/A_200_20-persistence/README.md), [서비스](A_200_seller/A_200_30-service/README.md), [API](A_200_seller/A_200_40-api/README.md) |

## 템플릿

- [A_XX_10 도메인 모델 템플릿](.template/A_XX_10-domain-model/domain-model.md)
- [A_XX_20 영속성 설계 템플릿](.template/A_XX_20-persistence/persistence-design.md)
- [A_XX_30 서비스 설계 템플릿](.template/A_XX_30-service/service-design.md)
- [A_XX_40 API 설계 템플릿](.template/A_XX_40-api/README.md)
- [A_XX_50 처리 시퀀스 템플릿](../80-sequence/.template/scenario.md)

## 예시 문서

- [주문 서비스 설계 예시](.examples/order/README.md)
