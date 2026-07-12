---
id: API.A.300-27
title: 운영자 인증 수동 처리 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-10
---

# API.A.300-27 운영자 인증 수동 처리

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/operator/auth/manual-actions` |
| operationId | `applyManualAuthAction` |
| 역할 | 승인된 잠금 해제, IdentityLink 해제·재연동 또는 단일 Session 폐기를 동기 실행한다. |
| API 유형 | Command |
| 인증 | 운영자 웹 Session, CSRF·Origin, 최근 strong auth |
| 권한 | `auth.case.execute`와 action별 세부 permission, 유효한 승인 |
| 노출 범위 | operator |
| 멱등성 | `Idempotency-Key` 필수이며 값 자체를 `operation_id`로 사용한다. |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, 미지원 중단 상태 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_27_apply_manual_auth_action.yaml](openapi/paths/API_A_300_27_apply_manual_auth_action.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

action·target 조합, 낙관적 version, HTTP 상태, 오류와 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [SD.A.30010 인증 도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- [SD.A.30020 인증 영속성 설계](../A_300_20-persistence/README.md)
- [SD.A.30030 인증 서비스 설계](../A_300_30-service/README.md)

## 책임과 경계

- 허용된 action을 승인 binding과 대상 version에 따라 한 번 실행한다.
- action별 permission을 분리하고 수동 DB 수정 기능을 제공하지 않는다.
- `revoke_sessions`는 `session` target 하나만 허용하며 사용자 전체 Session 일괄 폐기는 이 API의 책임이 아니다.
- 승인·증빙 원문은 복제하지 않고 접근 통제된 opaque reference만 저장한다.
- 사용자 계정 병합과 프로필·업무 데이터 수정은 제공하지 않는다.

## 보안과 개인정보

- 실행자 `user_id`, role과 assurance는 body가 아니라 검증된 운영자 Principal에서 가져온다.
- approval의 승인자 등급, 대상, action, 만료와 증빙 접근 권한을 외부 승인 시스템에서 확인한다.
- case·approval·evidence reference와 target ID는 일반 로그·metric label에 넣지 않는다.
- 고위험 action의 결과와 실행자·승인자는 Audit Context에 장기 보존한다.

| action | 추가 permission | 필수 승인 주체 |
| --- | --- | --- |
| `unlock_identity` | `auth.identity.unlock` | `platform_operator` |
| `revoke_identity_link` | `auth.identity_link.revoke` | `platform_operator` |
| `approve_relink` | `auth.identity_link.relink` | `platform_operator` |
| `revoke_sessions` | `auth.session.revoke` | `platform_operator` |

## 처리 규칙

1. 운영자 permission, strong auth, CSRF·Origin을 검증한다.
2. `Idempotency-Key`를 operation ID로 claim한다.
3. `auth.case.execute`, action별 permission, `platform_operator` 승인, action·target 조합, case, reason과 evidence reference를 검증한다.
4. target의 현재 row version이 `expectedTargetVersion`과 같은지 확인한다.
5. action을 동기 실행하고 상태 변경과 감사 Event를 저장한다.
6. 완료된 operation ID, action과 새 target version을 반환한다.

## 상태 변경과 트랜잭션

- target 상태 변경, 영향 Session 폐기, IdempotencyRecord와 OutboxEvent를 같은 트랜잭션에 저장한다.
- action마다 허용된 Aggregate만 변경하며 범위를 암묵적으로 넓히지 않는다.
- `revoke_sessions`는 요청한 Session Aggregate만 폐기하고 같은 사용자의 다른 Session으로 범위를 넓히지 않는다.
- 외부 승인 확인은 상태 변경 전 수행하고 결과 binding을 저장한다.
- 수동 작업 결과는 `completed`로 동기 반환하며 별도 polling 상태를 만들지 않는다.

## 멱등성과 동시성

- `Idempotency-Key` UUID를 `operation_id`로 사용해 별도 중복 식별자를 받지 않는다.
- 같은 operation ID와 같은 canonical 요청은 저장된 완료 결과를 반환한다.
- 같은 operation ID에 다른 target·action·approval이 오면 충돌로 거부한다.
- `expectedTargetVersion`이 현재 version과 다르면 상태를 바꾸지 않는다.
- Aggregate row lock과 action별 유일성 제약으로 동시 수동 처리를 직렬화한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 ProblemDetails 계약은 OpenAPI를 기준으로 한다.

- 승인 누락·만료·대상 불일치는 새 승인을 받은 뒤 다시 요청한다.
- target이 없거나 조회 권한이 없으면 민감한 존재 정보를 추가로 공개하지 않는다.
- target version이 바뀌었으면 상태를 다시 조회하고 새 operation ID로 재검토한다.
- 저장 실패는 완료로 기록하지 않으며 같은 operation ID의 안전한 재시도를 허용한다.

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command Handler | `ApplyManualIdentityOperationHandler`, `RevokeSessionHandler` |
| Aggregate / Entity | `Identity`, `IdentityLink`, `LoginFailure`, `Session` |
| Repository | action별 Aggregate Repository, `IdempotencyRepository` |
| Port | Approval verification port, Audit outbox port |
| Event | `EVT.A.300-11 인증 수단 수동 변경됨`, `EVT.A.300-26`, action별 감사 Event |

## 관측성과 운영

- operation ID, actor, action, 일반화된 target type, approval ref와 결과를 보안 감사에 남긴다.
- `auth_manual_action_total{action,result}`과 처리 지연을 관측한다.
- evidence 원문, Identity 표시값과 target ID는 일반 로그·trace·metric label에서 제외한다.
- 권한·승인·version 거부율과 반복 실패를 보안 경보로 관측한다.

## 검증 항목

- permission, strong auth, approval과 action binding을 모두 검증한다.
- action과 target type의 허용 조합 밖 요청을 거부한다.
- 각 action의 세부 permission과 `platform_operator` 승인을 교차 검증하고 `user` target Session 폐기를 거부한다.
- stale target version에서 어떤 상태도 바뀌지 않는다.
- 같은 operation ID 재시도에서 action과 감사 Event가 중복되지 않는다.
- 상태 변경과 Session 폐기, OutboxEvent가 원자적으로 저장된다.

## 연관 시퀀스

- 현재 연결된 다중 API 시퀀스는 없다.
- 선행 조회 API: `API.A.300-24`
- 승인 시스템, 운영자 API, Auth와 Audit Context를 잇는 수동 처리 시퀀스를 `80-sequence`에 추가한다.

## 호환성과 변경 정책

- 결과에 선택적 비민감 필드를 추가하는 변경은 하위 호환이다.
- action, target 의미와 승인 요구를 완화하는 변경은 보안 검토와 새 API 버전이 필요하다.
- 새 action은 permission, approval policy, version 규칙과 감사 Event를 함께 추가한다.

## 확인 필요

- 증빙 reference scheme, 사후 검토 상태와 감사 보존 기간을 확정한다.
