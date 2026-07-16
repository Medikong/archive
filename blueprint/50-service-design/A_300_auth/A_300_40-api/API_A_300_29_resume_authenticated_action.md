---
id: API.A.300-29
title: 인증 후 행동 복구 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-29 인증 후 행동 복구

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/intents/{intentId}/action-resume` |
| operationId | `resumeAuthenticatedAction` |
| 역할 | 로그인 전에 보존한 허용 action과 최소 context를 같은 Session에 한 번 전달한다. |
| API 유형 | Command |
| 인증 | Intent를 소비한 Session의 웹·모바일 공통 Bearer access JWT |
| 권한 | 현재 Session이 Intent의 `consumed_by_session_id`와 같아야 한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, 미지원 중단 상태 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_29_resume_authenticated_action.yaml](openapi/paths/API_A_300_29_resume_authenticated_action.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

허용 action별 `actionContext` oneOf, HTTP 상태, 오류와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [SD.A.30010 인증 도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- [SD.A.30020 인증 영속성 설계](../A_300_20-persistence/README.md)
- [SD.A.30030 인증 서비스 설계](../A_300_30-service/README.md)

## 책임과 경계

- 소비된 AuthenticationIntent의 allowlist action payload를 같은 Session에 한 번 전달한다.
- 지원 action은 구매 재개와 `seller_order_export`, `seller_member_manage`, `seller_account_update`, `seller_partnership_respond`다. 어떤 variant도 업무 명령 자체는 실행하지 않는다.
- 프론트엔드는 반환값을 검증한 뒤 Ingress를 통해 주문·드롭 API에 별도 멱등 key로 요청한다.
- seller action은 `sellerId`와 선택적 opaque `resourceId`만 전달한다. BFF는 이를 canonical `API.A.200-*` 호출로 변환하고 현재 membership·permission을 다시 확인한다.
- 임의 URL, script, 결제 비밀정보와 개인정보를 저장하거나 전달하지 않는다.

## 보안과 개인정보

- Istio가 access JWT와 active Session을 검증한다. Bearer 보호 API에는 CSRF 검사를 적용하지 않는다.
- Intent의 소비 Session과 현재 Session이 정확히 같아야 한다.
- action payload ciphertext는 응답 생성 중 메모리에서만 복호화한다.
- payload, Intent·Session ID와 멱등 key를 로그·trace·metric label에 넣지 않는다.

## 처리 규칙

1. 현재 Session과 UserAuthState를 확인한다.
2. Intent 상태, 만료와 `consumed_by_session_id` binding을 검증한다.
3. action 이름에 해당하는 allowlist schema로 암호화 payload를 다시 검증한다.
4. 최초 전달이면 delivered 상태와 감사 Event를 기록한다.
5. 내부 상대 복귀 경로와 최소 action context를 반환한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `SessionStatusProjection` | 사용 | 인증 뒤 작업을 재개하는 사용자의 Session 상태를 먼저 확인한다. |
| `AuthenticationIntent`, `ActionIntentPayload`, `IdempotencyRecord` | 우회 | intent 소유권, payload의 일회성 소비와 중복 재개 방지를 PostgreSQL 트랜잭션으로 확정한다. |
| action payload와 암호문 | 사용하지 않음 | 업무 요청 내용과 복호화 가능한 payload를 Redis에 복제하지 않는다. |

## 상태 변경과 트랜잭션

- 최초 성공 시 payload `delivered_at`, IdempotencyRecord와 감사 OutboxEvent를 같은 트랜잭션에 저장한다.
- payload 원문이나 ciphertext를 IdempotencyRecord와 OutboxEvent에 복제하지 않는다.
- 업무 Context Command는 이 트랜잭션에서 실행하지 않는다.
- replay TTL 종료 뒤 ciphertext를 crypto-shred한다.

## 멱등성과 동시성

- 멱등 범위는 API ID, Intent, 현재 Session, `Idempotency-Key`다.
- 같은 key와 같은 요청은 짧은 delivery replay TTL 동안 같은 action payload를 반환한다.
- 같은 key에 다른 요청이나 다른 key의 재소비는 거부한다.
- Intent와 payload row lock으로 동시에 들어온 최초 전달을 하나로 제한한다.
- replay TTL 뒤에는 payload를 다시 전달하지 않는다.

## 예외와 복구 규칙

정확한 HTTP 상태와 ErrorResponse 형식은 OpenAPI를 기준으로 한다.

- Intent가 만료됐거나 payload가 폐기됐으면 새 로그인 의도로 업무를 다시 시작한다.
- 소비 Session이 다르면 Intent 존재와 payload를 더 공개하지 않는다.
- 응답 유실은 replay TTL 안에서 같은 key로만 재시도한다.
- 필수 저장소 장애를 빈 action이나 성공 응답으로 바꾸지 않는다.

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command Handler | `DeliverAuthenticatedActionHandler` |
| Aggregate / Entity | `AuthenticationIntent`, `ActionIntentPayload`, `Session` |
| Repository | `AuthenticationIntentRepository`, action payload store, `IdempotencyRepository` |
| Adapter | action별 allowlist schema registry, payload encryption adapter |
| Event | `auth.action_resume.delivered` 감사 OutboxEvent |

## 관측성과 운영

- action 이름, 결과와 처리 지연만 집계한다.
- `auth_action_resume_total{action,result}`와 crypto-shred 지연을 관측한다.
- actionContext, 복귀 경로, Intent·Session ID는 관측 데이터에서 제외한다.
- replay TTL 종료 뒤 ciphertext 잔존 여부를 운영 점검한다.

## 검증 항목

- 다른 Session은 소비된 Intent를 전달받지 못한다.
- 허용되지 않은 action이나 actionContext 필드를 거부한다.
- 최초 전달, 같은 key replay와 다른 key 재소비를 구분한다.
- 전달 기록과 감사 OutboxEvent가 원자적으로 저장된다.
- replay TTL 뒤 ciphertext가 폐기되고 업무 명령은 자동 실행되지 않는다.

## 연관 시퀀스

- 현재 독립 시퀀스 문서는 없다.
- 선행 API: `API.A.300-01`과 로그인 또는 회원가입 완료 API
- 프론트엔드와 업무 Context까지 연결하는 인증 후 행동 재개 시퀀스는 후속으로 `A_300_50-sequence`에 추가한다.

## 호환성과 변경 정책

- 기존 action 결과에 선택적 비민감 필드를 추가하는 변경은 하위 호환이다.
- 새 action은 새 allowlist schema와 OpenAPI oneOf variant를 함께 추가한다.
- 기존 actionContext 필드 의미와 replay 규칙 변경은 새 API 버전에서 처리한다.

## 확인 필요

- seller action별 `resourceId` 필요 여부와 로그인 Intent 생성 화면의 최소 context를 consumer 호환성 테스트로 고정해야 한다.
- delivery replay TTL과 crypto-shred worker의 운영 기준을 확정한다.
