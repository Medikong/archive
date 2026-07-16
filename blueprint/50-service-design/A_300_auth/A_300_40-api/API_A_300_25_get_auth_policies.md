---
id: API.A.300-25
title: 인증 정책 조회 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-25 인증 정책 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/operator/auth/policies` |
| operationId | `getAuthPolicies` |
| 역할 | 현재 active 인증 정책 전체를 하나의 불변 version snapshot으로 조회한다. |
| API 유형 | Query |
| 인증 | `Authorization: Bearer <access-jwt>`, 최근 strong auth |
| 외부 인가 | 검증된 `auth.policy.read` 결정 |
| 노출 범위 | operator |
| 멱등성 | 해당 없음 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, 미지원 중단 상태 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_25_get_auth_policies.yaml](openapi/paths/API_A_300_25_get_auth_policies.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

정책 snapshot schema, HTTP 상태, 오류와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [SD.A.30010 인증 도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- [SD.A.30020 인증 영속성 설계](../A_300_20-persistence/README.md)
- [SD.A.30030 인증 서비스 설계](../A_300_30-service/README.md)

## 책임과 경계

- 로그인 잠금, Session TTL, refresh rotation, Verification과 Session 폐기 규칙을 같은 global version으로 반환한다.
- 과거 version 목록이나 diff 조회는 제공하지 않는다.
- signing key, pepper, provider secret과 내부 탐지 rule 원문은 반환하지 않는다.
- 정책 승인 절차와 배포는 운영 시스템 책임이며 이 Endpoint는 현재 snapshot만 조회한다.

## 보안과 개인정보

- access JWT의 Session, 외부 `auth.policy.read` 결정과 strong assurance를 모두 검증한다.
- Auth는 운영자의 role, permission, 판매자 소속 또는 업무 ACL을 조회하거나 저장하지 않는다.
- 정책 값은 개인정보를 포함하지 않지만 보안 설정이므로 외부 캐시와 body sampling을 금지한다.
- key ID, secret, 알고리즘 내부 parameter와 탐지 우회에 악용될 수 있는 값을 제외한다.
- 조회 감사에는 actor와 policy version만 남긴다.

## 처리 규칙

1. access JWT의 Session, 외부 `auth.policy.read` 결정과 strong assurance를 확인한다.
2. active `auth_policy_versions` 행과 모든 하위 rule을 같은 version으로 읽는다.
3. 누락되거나 다른 version인 하위 rule이 있으면 정상 snapshot으로 반환하지 않는다.
4. 운영 가능한 공개 필드만 정책별 구조로 변환한다.
5. global version과 효력 시각을 포함해 반환한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| 운영자 `SessionStatusProjection` | 사용 | 운영자 요청을 처리하기 전에 Session 상태를 확인한다. 권한 판정은 상위 운영자 인가 계층이 담당한다. |
| `AuthenticationPolicySnapshotProjection` | 사용 | 정책은 변경 빈도보다 조회 빈도가 높으므로 인가가 끝난 뒤 활성 version의 불변 snapshot을 반환한다. |
| HTTP 응답 | 사용하지 않음 | 내부 Redis snapshot을 사용하더라도 운영자 응답에는 `Cache-Control: no-store`를 유지한다. |

## 상태 변경과 트랜잭션

- 상태를 바꾸지 않는 Query다.
- 하나의 active version에 속한 parent와 하위 rule을 일관된 snapshot으로 읽는다.
- 부분 정책이나 섞인 version을 성공 형태로 반환하지 않는다.

## 멱등성과 동시성

- GET Query이므로 Idempotency-Key를 받지 않는다.
- 응답의 global version은 후속 정책 변경의 `If-Match` 기준이다.
- 조회 직후 정책이 교체될 수 있으므로 변경 Command가 version을 다시 검증한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 ErrorResponse 형식은 OpenAPI를 기준으로 한다.

- 외부 인가 결정이 거부되면 정책 값과 존재 여부를 추가로 공개하지 않는다.
- 인가 결정이 만료됐거나 strong assurance가 끝났으면 재승인 또는 재인증 뒤 다시 조회한다.
- active 정책이나 필수 하위 rule이 불완전하면 빈 정책이 아니라 서비스 장애로 처리한다.
- 클라이언트는 장애 시 이전 응답을 임의로 active 정책으로 간주하지 않는다.

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Handler | `GetSessionPolicyHandler` |
| Value Object | `LoginLockPolicy`, `TokenTtlPolicy`, `RefreshRotationPolicy`, `VerificationPolicy`, `SessionRevocationPolicy` |
| Repository | `PolicyRepository.FindActive` |
| Read Model | `RM.A.300-03 세션 정책 조회` |
| External Authorization | 검증된 `auth.policy.read` 결정 proof |

## 관측성과 운영

- 조회 결과와 global policy version만 구조화 로그에 기록한다.
- `auth_policy_query_total{result}`와 저장소 지연을 관측한다.
- 정책 전체 body는 로그와 trace에 복제하지 않는다.
- active snapshot 불일치는 즉시 운영 경보 대상으로 둔다.

## 검증 항목

- 모든 정책 값과 하위 rule이 같은 global version인지 검증한다.
- secret과 내부 탐지 rule 원문이 응답에 포함되지 않는다.
- 유효한 외부 인가 결정 또는 strong assurance가 없는 Principal은 정책을 읽지 못한다.
- active 정책 부재와 일부 rule 유실을 정상 빈 목록으로 반환하지 않는다.
- 응답 version을 정책 변경 `If-Match`에 사용할 수 있다.

## 연관 시퀀스

- 현재 연결된 다중 API 시퀀스는 없다.
- 후속 변경 API: `API.A.300-26`
- 정책 승인·조회·변경 절차가 확정되면 운영자 정책 시퀀스를 `A_300_50-sequence`에 추가한다.

## 호환성과 변경 정책

- 정책 object에 선택 필드를 추가하는 변경은 하위 호환이다.
- global version 의미와 시간 단위 변경은 새 API 버전에서 처리한다.
- 새 정책 묶음도 기존 global snapshot 원칙을 유지한다.

## 확인 필요

- 과거 policy version과 변경 diff 조회 API가 필요한지 운영 요구를 확인한다.
- 정책 조회 자체의 감사 보존 기간과 alert 기준을 확정한다.
