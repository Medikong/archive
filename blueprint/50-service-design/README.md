---
id: service-design-index
title: 서비스 상세 설계 인덱스
type: service-design-index
status: active
tags: [service-design, bounded-context, domain-model, persistence, service, api]
source: local
created: 2026-07-06
updated: 2026-07-11
---

# 서비스 상세 설계 인덱스

## 역할

이 폴더는 이벤트스토밍에서 분해한 바운디드 컨텍스트별 상세 설계를 모은다. MVP 전체 이벤트스토밍 문서 하나를 그대로 서비스 설계로 옮기지 않고, `Context 인증`, `Context 쿠폰`처럼 책임 경계가 나뉜 단위마다 도메인 모델, 영속성, 서비스, API 설계를 진행한다.

## 폴더 구조

```text
50-service-design/
  <context>/
    README.md
    <context_id>_10-domain-model/
    <context_id>_20-persistence/
    <context_id>_30-service/
    <context_id>_40-api/
```

## 하위 설계 번호

| Suffix | 이름 | 역할 |
| --- | --- | --- |
| `10` | domain-model | Aggregate, Entity, Value Object, Domain Event, Policy, Business Rule 설계 |
| `20` | persistence | Repository, Database Schema, Index, Migration, 저장 경계 설계 |
| `30` | service | Application Service, Domain Service, Command Handler, Transaction Boundary 설계 |
| `40` | api | REST API, 이벤트 계약, 요청/응답, 오류 계약 설계 |

## 작성 단위

- `40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md`처럼 전체 MVP를 다루는 문서는 서비스 설계 폴더 하나로 옮기지 않는다.
- 전체 이벤트스토밍에서 식별한 바운디드 컨텍스트를 다시 분해해 `A_300_auth/`, `A_19_coupon/`, `<context_id>_<context_name>/` 같은 단위로 설계한다.
- 설계 분야 하나가 커지면 해당 번호 폴더 안에서 파일을 여러 개로 나눈다.
- 도메인 모델 폴더 안에서 Aggregate, Entity, Value Object 문서를 반드시 분리하지 않는다. Aggregate 단위 문서 안에 Entity와 Value Object를 함께 정의할 수도 있고, 작으면 단일 도메인 모델 문서 하나로 끝낼 수도 있다.
- 같은 컨텍스트의 설계 문서는 서로 가까운 상대 경로로 연결하고, 요구사항/UC/BC/시나리오는 원천 문서 링크로 연결한다.

## 식별자 규칙

서비스 상세 설계는 `SD` prefix를 사용한다. 컨텍스트 루트 문서는 기존 업무 번호를 그대로 쓰고, 하위 설계 영역은 업무 번호 뒤에 `10`, `20`, `30`, `40`을 붙인다.

| 대상 | 예시 | 설명 |
| --- | --- | --- |
| Context 인증 루트 | `SD.A.300` | `A_300_auth/README.md` |
| Context 인증 도메인 모델 | `SD.A.30010` | `A_300_auth/A_300_10-domain-model/SD_A_30010_auth_domain_model.md` |
| Context 인증 영속성 | `SD.A.30020` | `A_300_auth/A_300_20-persistence/README.md` |
| Context 인증 서비스 | `SD.A.30030` | `A_300_auth/A_300_30-service/README.md` |
| Context 인증 API | `SD.A.30040` | `A_300_auth/A_300_40-api/README.md` |

폴더명은 컨텍스트 식별자와 하위 설계 번호를 함께 드러낸다. 예를 들어 Context 인증 도메인 모델 영역은 `A_300_10-domain-model/`이고, 문서 식별자는 `SD.A.30010`이다.

## 현재 컨텍스트

| 컨텍스트 폴더 | 바운디드 컨텍스트 | 상태 | 주요 근거 |
| --- | --- | --- | --- |
| [A_01_user](A_01_user/README.md) | Context 사용자 | draft | [BC.A.01](../40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md), [REQ.A.05](../00-requirements/REQ_A_05_auth_member.md), [도메인 모델](A_01_user/A_01_10-domain-model/README.md), [영속성](A_01_user/A_01_20-persistence/README.md), [서비스](A_01_user/A_01_30-service/README.md), [API](A_01_user/A_01_40-api/README.md) |
| [A_300_auth](A_300_auth/README.md) | Context 인증 | draft | [BC.A.300](../40-event-storming-bounded-context/BC_A_300_auth_member.md), [BC.A.01](../40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md) |
| [A_19_coupon](A_19_coupon/README.md) | Context 쿠폰 | draft | [BC.A.19](../40-event-storming-bounded-context/BC_A_19_coupon.md), [REQ.A.02](../00-requirements/REQ_A_02_coupon_benefit.md), [도메인 모델](A_19_coupon/A_19_10-domain-model/README.md), [영속성](A_19_coupon/A_19_20-persistence/README.md), [서비스](A_19_coupon/A_19_30-service/README.md), [API](A_19_coupon/A_19_40-api/README.md) |

## 템플릿

- [A_XX_10 도메인 모델 템플릿](.template/A_XX_10-domain-model/domain-model.md)
- [A_XX_20 영속성 설계 템플릿](.template/A_XX_20-persistence/persistence-design.md)
- [A_XX_30 서비스 설계 템플릿](.template/A_XX_30-service/service-design.md)
- [A_XX_40 API 설계 템플릿](.template/A_XX_40-api/README.md)

## 예시 문서

- [주문 서비스 설계 예시](.examples/order/README.md)
