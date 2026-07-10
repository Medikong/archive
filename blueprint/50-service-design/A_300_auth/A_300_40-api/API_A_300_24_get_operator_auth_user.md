---
id: API.A.300-24
title: 운영자 인증 상태 조회 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-10
---

# API.A.300-24 운영자 인증 상태 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/operator/auth/users/{userId}` |
| operationId | `getOperatorAuthUser` |
| 역할 | CS 처리를 위해 사용자 인증 상태, Identity별 잠금·Link와 활성 Session 수를 조회한다. |
| API 유형 | Query |
| 인증 | 운영자 웹 Session, 최근 strong auth |
| 권한 | 현재 AccessGrant의 `auth.case.read`, 필수 조회 사유 |
| 노출 범위 | operator |
| 멱등성 | 해당 없음 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, 미지원 중단 상태 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_24_get_operator_auth_user.yaml](openapi/paths/API_A_300_24_get_operator_auth_user.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

필수 감사 사유 헤더, 응답 schema, HTTP 상태, 오류와 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [SD.A.30010 인증 도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- [SD.A.30020 인증 영속성 설계](../A_300_20-persistence/README.md)
- [SD.A.30030 인증 서비스 설계](../A_300_30-service/README.md)

## 책임과 경계

- UserAuthState, Identity·IdentityLink, 로그인 잠금과 Session 요약을 하나의 운영자 Read Model로 제공한다.
- 잠금은 사용자 전체 값으로 합치지 않고 각 Identity에 적용된 상태로 반환한다.
- credential, Challenge proof, token과 비밀번호 hash는 조회하거나 반환하지 않는다.
- 프로필, 주문, 구매 제한과 고객 지원 증빙 원문은 다른 Context 책임이다.

## 보안과 개인정보

- 운영자 Principal의 현재 AccessGrant, `auth.case.read`, strong assurance와 필수 조회 사유를 모두 확인한다.
- Identity 표시값은 권한 있는 운영자에게도 마스킹하며 unmask 기능은 제공하지 않는다.
- 대상 부재와 권한 부족의 공개 범위는 운영자 정보 노출 정책을 따른다.
- 조회자 `user_id`, 대상 `user_id`, 사유 코드와 결과를 감사하되 Identity 원문을 남기지 않는다.

## 처리 규칙

1. 운영자 Session, 현재 AccessGrant, permission, strong assurance와 감사 사유 형식을 검증한다.
2. 대상 UserAuthState와 인증 상태 Read Model을 조회한다.
3. Identity별 검증·Link·잠금 상태와 active Session 수를 구성한다.
4. 표시값을 마스킹하고 민감 필드를 제거한다.
5. 조회 감사 이벤트를 기록한 뒤 응답한다.

## 상태 변경과 트랜잭션

- 인증 업무 상태를 바꾸지 않는 Query다.
- 보안 감사 기록만 별도 append 또는 OutboxEvent로 저장한다.
- 감사 기록 실패를 무시하고 민감 조회 결과를 성공으로 반환하지 않는다.
- Query 중 Session·Identity·잠금 상태를 정정하거나 해제하지 않는다.

## 멱등성과 동시성

- GET Query이므로 Idempotency-Key를 받지 않는다.
- 각 항목의 조회 시점 상태를 반환하며 후속 변경 가능성을 보장하지 않는다.
- 운영자 변경 Command는 이 응답의 version을 사용해 별도 낙관적 동시성 검사를 수행한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 ProblemDetails 계약은 OpenAPI를 기준으로 한다.

- permission이 없으면 대상 존재 여부를 더 공개하지 않는다.
- AccessGrant가 폐기·만료됐거나 strong assurance가 끝났으면 재승인 또는 재인증 뒤 다시 조회한다.
- 대상 인증 Read Model이 없으면 운영자 전용 not-found 오류를 반환한다.
- 필수 감사 저장소 또는 인증 저장소 장애를 빈 정상 응답으로 바꾸지 않는다.

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Handler | `GetAuthenticationStatusHandler` |
| Aggregate / Entity | `UserAuthState`, `Identity`, `IdentityLink`, `LoginFailure`, `Session` |
| Read Model | `RM.A.300-02 인증 상태 조회` |
| Authorization | `AccessGrant`, `auth.case.read` |
| Audit | 운영자 인증 상태 조회 감사 OutboxEvent |

## 관측성과 운영

- 운영자 `user_id`, 사유 코드, 결과와 지연을 접근 통제된 감사 데이터로 기록한다.
- `auth_operator_user_query_total{result}`와 조회 지연을 관측한다.
- 대상 ID와 Identity 표시값은 metric label과 일반 trace에 넣지 않는다.
- 응답 body sampling과 중간 캐시를 비활성화한다.

## 검증 항목

- 현재 AccessGrant, permission, strong assurance와 감사 사유가 모두 있어야 조회된다.
- Identity별 잠금 상태가 다른 Identity에 합쳐지지 않는다.
- restricted/deactivated UserAuthState를 active로 완화하지 않는다.
- 표시값이 항상 마스킹되고 secret 계열 필드가 응답에 없다.
- 조회 행위의 감사 기록과 실패 정책을 검증한다.

## 연관 시퀀스

- 현재 연결된 다중 API 시퀀스는 없다.
- 후속 운영자 처리: `API.A.300-27`
- 운영 절차가 확정되면 상태 조회와 수동 처리 시퀀스를 `80-sequence`에 추가한다.

## 호환성과 변경 정책

- Identity 요약의 선택 필드 추가는 하위 호환이다.
- 마스킹 규칙, UserAuthState 의미와 permission 이름 변경은 보안 검토와 버전 변경을 거친다.
- unmask 기능은 이 Endpoint 확장이 아니라 별도 승인 API로 설계한다.

## 확인 필요

- 운영자 대상 없음 오류를 보여 줄 수 있는 역할 범위를 확정한다.
- 감사 사유 코드 목록과 감사 저장 실패 시 운영 정책을 확정한다.
