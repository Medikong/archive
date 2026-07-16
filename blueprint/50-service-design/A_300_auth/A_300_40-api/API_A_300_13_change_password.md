---
id: API.A.300-13
title: 비밀번호 변경 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-13 비밀번호 변경

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `PUT /api/v1/auth/password-resets/{passwordResetId}/password` |
| operationId | `completePasswordReset` |
| 역할 | 검증된 reset 권한으로 비밀번호를 교체하고 기존 Session을 폐기한다. |
| API 유형 | Command |
| 인증 | `credentialDelivery=web_auth_flow`의 웹 auth-flow cookie 또는 `credentialDelivery=mobile_reset_grant`의 모바일 auth flow token과 reset grant |
| 권한 | PasswordReset, 채널, 목적과 단일 사용 reset 권한의 binding 일치 |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_13_change_password.yaml](openapi/paths/API_A_300_13_change_password.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

채널별 request schema와 `credentialDelivery` discriminator, `x-channel-body-binding`, 응답 header, HTTP 상태, 오류 body와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [SCN.A.310-01](../A_300_50-sequence/SCN_A_310_01_password_reset.md) |

## 책임과 경계

- 새 비밀번호 정책과 확인값을 검증하고 PasswordCredential을 교체한다.
- PasswordReset과 reset grant를 소비하고 대상 사용자의 전체 Session과 refresh family를 폐기한다.
- MVP에서는 비밀번호 변경 뒤 새 Session을 자동 발급하지 않는다.
- 프로필 변경이나 사용자 계정 상태 복구는 수행하지 않는다.

## 보안과 개인정보

- 웹은 `credentialDelivery=web_auth_flow`과 auth-flow cookie·CSRF token·Origin을 함께 검증하고 reset grant를 허용하지 않는다.
- 모바일은 `credentialDelivery=mobile_reset_grant`, auth flow token과 body의 reset grant를 모두 검증한다.
- 비밀번호와 reset grant 원문을 저장·로그·trace·metric·감사 event에 남기지 않는다.
- 비밀번호 정책 오류는 공개 가능한 정책 항목만 반환하고 기존 credential 정보를 공개하지 않는다.
- 성공 시 웹 auth-flow cookie를 만료시키는 `Set-Cookie`를 반환하고 모바일 auth flow token도 더 이상 사용할 수 없게 한다.

## 처리 규칙

1. `credentialDelivery`가 제출한 보안 credential의 채널과 일치하는지 확인한 뒤 PasswordReset 상태와 채널별 reset 권한을 검증한다.
2. 새 비밀번호와 확인값의 일치 및 현재 PasswordPolicy를 검증한다.
3. 기존 PasswordCredential을 `replaced`로 닫고 새 credential version을 만든다.
4. PasswordReset과 연결된 AuthenticationIntent를 완료 사유로 소비한다.
5. 해당 `user_id`의 모든 Session과 refresh family를 폐기하고 자동 로그인 없이 완료한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `PasswordReset`, `PasswordCredential`, `Session`, `IdempotencyRecord` | 우회 | reset grant 소비, 비밀번호 교체, 기존 Session 폐기를 PostgreSQL 트랜잭션으로 함께 확정해야 한다. |
| `AuthenticationPolicySnapshotProjection` | 사용 | 비밀번호 강도와 credential 정책을 활성 정책 version 기준으로 검사한다. |
| `SessionStatusProjection` | 무효화 | 커밋 뒤 사용자의 모든 기존 Session을 삭제하지 않고 `revoked` 상태로 기록해 남은 access JWT도 즉시 거부한다. |

## 상태 변경과 트랜잭션

- 시작 상태는 `PasswordReset.challenge_verified`다.
- PasswordCredential 교체, PasswordReset `completed`, reset grant 소비, 전 Session 폐기와 감사 OutboxEvent를 한 트랜잭션에 저장한다.
- 성공 후 reset grant hash를 제거하고 완료 시각만 보존한다.
- 어느 단계든 실패하면 기존 PasswordCredential과 Session 상태를 유지한다.
- 외부 호출 없이 로컬 transaction으로 완료한다.

## 멱등성과 동시성

- 멱등 범위는 API ID, PasswordReset과 `Idempotency-Key`다.
- 비밀번호와 grant 원문 대신 canonical request의 keyed HMAC fingerprint만 저장한다.
- 같은 key와 같은 요청은 이미 완료된 상태를 `204`로 재생한다.
- 같은 key와 다른 요청은 충돌로 거부하며 새 PasswordCredential을 만들지 않는다.
- PasswordReset row lock과 credential version으로 완료와 grant 소비를 한 번만 허용한다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ErrorResponse schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| reset 권한 불일치 | 세부 binding과 계정 상태를 숨긴다. | 재설정을 처음부터 다시 시작한다. |
| reset grant 만료 | Challenge 실패와 구분되는 만료 code만 공개한다. | 새 Challenge로 소유를 다시 확인한다. |
| 비밀번호 정책 미충족 | 공개 가능한 정책만 제공한다. | 비밀번호를 수정해 같은 작업을 새 key로 요청한다. |
| transaction 실패 | 성공 형태를 반환하지 않는다. | 상태를 다시 확인한 뒤 같은 key로 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `CompletePasswordResetHandler` |
| Aggregate / Entity | `PasswordReset`, `PasswordCredential`, `Session`, `AuthenticationIntent` |
| Repository / Read Model | PasswordReset·Credential·Session Repository, `IdempotencyRecord` |
| Port / Adapter | PasswordHasherPort |
| Domain / Integration Event | 비밀번호 변경·Session 폐기 감사 OutboxEvent |

## 관측성과 운영

- 로그에는 password reset ID, session 폐기 건수와 일반화된 결과만 남긴다.
- 비밀번호 변경 성공·정책 실패·grant 만료와 Session 폐기 지연을 관측한다.
- PasswordReset lock, credential 교체, Session 폐기와 outbox 저장을 transaction span으로 묶는다.
- request/response body와 secret은 sampling하지 않는다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 웹과 모바일 request variant가 채널별 reset 권한과 일치한다.
- 웹 variant는 reset grant를 거부하고 모바일 variant는 reset grant 누락을 거부한다.
- 성공한 웹 응답이 auth-flow cookie 만료 header를 포함한다.
- credential 교체 실패 시 기존 비밀번호와 Session을 사용할 수 있다.
- 성공 후 기존 모든 Session과 refresh family가 폐기된다.
- 동시 요청과 같은 key 재요청이 credential을 중복 생성하지 않는다.
- 비밀번호와 grant가 로그, trace, metric label, event와 IdempotencyRecord에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.310-01 비밀번호 재설정](../A_300_50-sequence/SCN_A_310_01_password_reset.md)
- 관련 API: `API.A.300-10`, `API.A.300-11`, `API.A.300-12`
- 여러 참여자의 Mermaid 다이어그램은 `A_300_50-sequence` 문서에서 관리한다.

## 호환성과 변경 정책

- 공개 가능한 비밀번호 정책 필드 추가는 하위 호환으로 처리한다.
- reset 권한 전달 방식과 성공 후 자동 로그인 정책 변경은 새 버전에서 제공한다.
- 이전 PasswordPolicy는 발급 시각이 아니라 완료 요청 시 적용하는 현재 규칙을 유지한다.

## 확인 필요

- PasswordPolicy의 초기 길이·복잡도와 과거 비밀번호 재사용 제한값을 확정해야 한다.
