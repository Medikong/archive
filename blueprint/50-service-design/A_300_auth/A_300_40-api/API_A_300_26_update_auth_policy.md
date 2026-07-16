---
id: API.A.300-26
title: 인증 정책 변경 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-26 인증 정책 변경

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `PATCH /api/v1/operator/auth/policies/{policyName}` |
| operationId | `updateAuthPolicy` |
| 역할 | 선택한 정책 묶음을 변경하고 나머지 값을 복제해 새 global 정책 snapshot을 활성화한다. |
| API 유형 | Command |
| 인증 | `Authorization: Bearer <access-jwt>`, 최근 strong auth |
| 외부 인가 | 검증된 `auth.policy.write` 결정 |
| 노출 범위 | operator |
| 멱등성 | `Idempotency-Key`와 global version `If-Match` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, 미지원 중단 상태 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_26_update_auth_policy.yaml](openapi/paths/API_A_300_26_update_auth_policy.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

path·body `policyName` binding, 정책별 patch schema, 안전 상한, global ETag, 오류와 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [SD.A.30010 인증 도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- [SD.A.30020 인증 영속성 설계](../A_300_20-persistence/README.md)
- [SD.A.30030 인증 서비스 설계](../A_300_30-service/README.md)

## 책임과 경계

- 허용된 policyName과 필드만 안전 범위 안에서 변경한다.
- body의 필수 `policyName`은 path 값과 정확히 같아야 하며 discriminator로 patch variant를 선택한다.
- 변경 대상 외 현재 정책 값을 복제해 완전한 새 global snapshot을 만든다.
- `verification.rules`와 `session-revocation.rules`는 해당 정책의 기존 규칙 전체를 교체하며 부분 병합하지 않는다.
- secret, signing key, pepper와 Provider credential 변경에는 사용하지 않는다.
- 팀 승인과 변경 티켓 절차는 운영 시스템 책임이며 이 API는 검증된 Principal과 사유를 받는다.

## 보안과 개인정보

- access JWT의 Session, 외부 `auth.policy.write` 결정과 strong auth를 모두 확인한다.
- Bearer 보호 API에는 웹 refresh cookie를 사용하지 않으므로 CSRF·Origin 검사를 적용하지 않는다.
- Auth는 운영자의 role, permission, 판매자 소속 또는 업무 ACL을 조회하거나 저장하지 않는다.
- actor는 body에서 받지 않고 검증된 Principal의 `user_id`를 사용한다.
- changeReason에 개인정보와 secret을 허용하지 않는다.
- 정책 전체와 변경 전후 값을 일반 로그·trace body에 기록하지 않는다.

## 처리 규칙

1. 외부 `auth.policy.write` 결정, strong auth와 global `If-Match`를 확인한다.
2. path와 body의 `policyName` 일치, discriminator variant, 허용된 patch 필드와 OpenAPI 안전 상한을 검증한다.
3. 현재 active snapshot을 읽고 대상 필드만 변경한 새 전체 snapshot을 구성한다.
4. 병합된 Session TTL의 `webIdle <= webAbsolute`, `mobileAccess < mobileRefresh <= rememberMe`, Verification의 목적·채널 허용 조합과 `(purpose, channel)` 유일성, Session revocation의 trigger 유일성과 password reset 필수 scope를 재검증한다.
5. 새 version을 active로 만들고 이전 version을 superseded로 닫는다.
6. 새 global version과 ETag를 반환한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| 운영자 `SessionStatusProjection` | 사용 | 정책을 바꾸려는 운영자의 Session 상태를 먼저 확인한다. 권한 판정은 상위 운영자 인가 계층이 담당한다. |
| PostgreSQL 정책 모델, `If-Match` version, `IdempotencyRecord` | 우회 | 동시 수정 충돌과 새 정책 version 생성은 PostgreSQL 원장에서만 확정한다. |
| `AuthenticationPolicySnapshotProjection` | 갱신 | 커밋 뒤 새 version의 불변 snapshot을 먼저 기록한 다음 활성 version pointer를 교체해 부분 갱신을 노출하지 않는다. |

## 상태 변경과 트랜잭션

- 새 parent snapshot, 전체 교체된 하위 rule, 이전 snapshot 종료, IdempotencyRecord와 감사 OutboxEvent를 같은 트랜잭션에 저장한다.
- 일부 정책만 새 version으로 저장하거나 parent와 하위 rule version을 섞지 않는다.
- 이미 발급된 artifact TTL은 소급해 늘리지 않는다.
- 긴급 단축의 기존 Session 처리에는 별도 Session 폐기 정책을 적용한다.

## 멱등성과 동시성

- 멱등 범위는 API ID, 운영자 `user_id`, `Idempotency-Key`다.
- 같은 key·같은 요청은 이미 생성한 global version과 ETag를 반환한다.
- replay 판정은 stale `If-Match` 검사보다 먼저 수행해 정상 재시도를 허용한다.
- 같은 key에 다른 patch는 충돌로 거부한다.
- 새 실행은 global `If-Match`와 active version이 다르면 변경하지 않는다.

## 예외와 복구 규칙

정확한 HTTP 상태와 ErrorResponse 형식은 OpenAPI를 기준으로 한다.

- 안전 범위를 벗어난 값이나 불완전한 rule 집합은 반영하지 않는다.
- path와 body의 `policyName`이 다르거나 배열에 일부 규칙만 보내면 반영하지 않는다.
- TTL 대소 관계, Verification 목적·채널 조합, rule 업무 키 유일성 또는 password reset의 `user_sessions + refresh_family` 필수 범위가 깨지면 전체 snapshot을 거부한다.
- global version이 달라졌으면 최신 정책을 조회하고 변경안을 다시 검토한다.
- 저장이나 감사 OutboxEvent 생성 실패 시 이전 정책을 active로 유지한다.

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command Handler | `ChangeAuthenticationPolicyHandler` |
| Value Object | `LoginLockPolicy`, `TokenTtlPolicy`, `RefreshRotationPolicy`, `VerificationPolicy`, `SessionRevocationPolicy` |
| Repository | `PolicyRepository.Activate`, `IdempotencyRepository` |
| External Authorization | 검증된 `auth.policy.write` 결정 proof, strong assurance |
| Event | `EVT.A.300-30 인증 정책 변경됨`, 감사 OutboxEvent |

## 관측성과 운영

- actor, global old/new version, policyName, 사유 코드와 결과를 감사 기록에 남긴다.
- `auth_policy_change_total{policy_name,result}`와 처리 지연을 관측한다.
- 상세 정책 값은 metric label과 일반 trace에 넣지 않는다.
- 사전조건 충돌과 안전 범위 거부율을 운영 경보로 관측한다.

## 검증 항목

- 외부 인가 결정 또는 strong auth가 없으면 변경되지 않는다.
- path·body `policyName` 불일치, policyName별 허용 필드 외 입력과 unsafe 값을 거부한다.
- 규칙 배열을 전체 교체하고 이전 규칙이 암묵적으로 남지 않는지 검증한다.
- 병합된 TTL 대소 관계, 목적별 Verification channel, rule 업무 키 유일성과 password reset 필수 폐기 범위를 검증한다.
- 새 snapshot의 parent와 모든 하위 rule이 같은 version이다.
- 같은 key replay와 stale global version 경합에서 snapshot이 중복 생성되지 않는다.
- 정책 저장과 감사 Event가 원자적으로 처리된다.

## 연관 시퀀스

- 현재 연결된 다중 API 시퀀스는 없다.
- 선행 조회 API: `API.A.300-25`
- 승인·변경·감사 절차가 확정되면 운영자 정책 변경 시퀀스를 `A_300_50-sequence`에 추가한다.

## 호환성과 변경 정책

- 기존 policyName에 선택 patch 필드를 추가하는 변경은 하위 호환이다.
- 필드 단위, global version 의미와 ETag 형식 변경은 새 API 버전에서 처리한다.
- 새 정책 묶음도 전체 snapshot 생성과 global version 검사를 유지한다.

## 확인 필요

- 문서 상한보다 더 좁은 환경별 운영 한계와 긴급 변경 승인 등급을 확정한다.
- 즉시 적용만 지원할지 예약된 `effectiveAt`을 후속으로 지원할지 결정한다.
