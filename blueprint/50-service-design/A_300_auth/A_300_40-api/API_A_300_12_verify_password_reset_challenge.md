---
id: API.A.300-12
title: 비밀번호 재설정 Challenge 검증 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-12 비밀번호 재설정 Challenge 검증

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/password-resets/{passwordResetId}/challenges/{challengeId}/verify` |
| operationId | `verifyPasswordResetChallenge` |
| 역할 | 이메일 또는 SMS code를 검증하고 채널별 reset 권한을 발급한다. |
| API 유형 | Command |
| 인증 | 웹 사전 인증 cookie 또는 모바일 auth flow token으로 PasswordReset과 Challenge 소유 증명 |
| 권한 | PasswordReset, Challenge, 소유 컨텍스트와 채널 binding 일치 |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_12_verify_password_reset_challenge.yaml](openapi/paths/API_A_300_12_verify_password_reset_challenge.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

공용 `VerificationCode` request schema, 채널별 response와 `credentialDelivery` discriminator, 응답 header, HTTP 상태, 오류 body와 wire 예시는 OpenAPI를 기준으로 한다.

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

- code proof를 검증하고 PasswordReset을 `challenge_verified`로 전환한다.
- 웹은 기존 auth-flow cookie에 reset 권한을 바인딩하고 모바일만 1회용 reset grant를 전달한다.
- 비밀번호 교체와 기존 Session 폐기는 이 Endpoint에서 수행하지 않는다.
- 이메일과 SMS 모두 이번 버전에서는 code proof만 지원한다.

## 보안과 개인정보

- 웹은 auth-flow cookie, CSRF token, Origin을 함께 검증하고 모바일은 auth flow token을 검증한다.
- code와 reset grant 원문은 DB, 일반 로그, trace, metric label과 IdempotencyRecord에 저장하지 않는다.
- decoy Challenge 실패를 실제 Challenge의 실패와 같은 공개 오류로 변환한다.
- 같은-key 모바일 성공 재요청은 reset grant hash를 원자적으로 새 값으로 교체하고 새 grant만 반환한다. grant 원문이나 암호화 replay payload는 저장하지 않는다.

## 처리 규칙

1. PasswordReset, Challenge, 목적, 채널과 소유 컨텍스트 binding을 확인한다.
2. Challenge 상태, TTL과 시도 제한을 확인하고 code proof를 검증한다.
3. 성공 시 Challenge를 소비하고 PasswordReset을 `challenge_verified`로 바꾼다.
4. 웹은 reset 권한을 auth-flow 컨텍스트에 바인딩하고 모바일은 1회용 reset grant를 만든다.
5. `credentialDelivery`에 맞는 웹 또는 모바일 응답을 반환한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `PasswordReset`, `VerificationChallenge`, `IdempotencyRecord` | 우회 | challenge의 일회성 소비, 실패 횟수 증가, reset grant 교체를 row lock과 트랜잭션으로 확정해야 한다. |
| `AuthenticationPolicySnapshotProjection` | 사용 | 검증 횟수와 reset grant 수명 같은 활성 `VerificationPolicy`를 일관되게 적용한다. |
| reset grant와 challenge code | 사용하지 않음 | 탈취 시 비밀번호 변경 권한이 되는 proof와 code를 Redis 조회값으로 만들지 않는다. |

## 상태 변경과 트랜잭션

- 시작 상태는 `PasswordReset.requested`와 `VerificationChallenge.issued`다.
- Challenge 검증·소비, PasswordReset 상태 변경, reset grant hash와 감사 OutboxEvent를 한 트랜잭션에 저장한다.
- 같은-key 모바일 성공 재요청은 같은 PasswordReset과 IdempotencyRecord를 잠그고 reset grant hash를 교체한다. 이전 grant는 즉시 무효화한다.
- 실패 시 attempt count와 정책상 terminal 상태를 같은 트랜잭션에서 반영한다.
- 성공 종료 상태는 `PasswordReset.challenge_verified`와 `VerificationChallenge.verified`다.

## 멱등성과 동시성

- 멱등 범위는 API ID, Challenge, 소유 컨텍스트와 `Idempotency-Key`다.
- code 원문 대신 canonical request의 keyed HMAC fingerprint를 저장한다.
- 같은 key는 전송 결과가 불확실한 동일 request body의 재전송에만 사용하며, 이전 실패를 재생하고 attempt count를 다시 늘리지 않는다.
- 검증 실패 뒤 새로 입력하거나 수정한 code는 새 `Idempotency-Key`로 제출한다. 기존 key와 다른 body는 멱등 충돌로 거부한다.
- 같은 key와 같은 모바일 성공은 동일 PasswordReset을 유지한 채 새 reset grant를 발급하고 이전 grant를 즉시 무효화한다.
- Challenge row lock과 PasswordReset version으로 성공 소비를 한 번만 허용한다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ErrorResponse schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| code 불일치 | 대상 존재 여부와 남은 내부 횟수를 숨긴다. | 전송 불확실성은 같은 key·같은 body로 재시도하고, 새로 입력하거나 수정한 code는 새 key로 제출한다. |
| Challenge 만료 | 연결 대상과 내부 상태를 공개하지 않는다. | 새 Challenge를 발급한다. |
| PasswordReset binding 불일치 | 세부 원인을 일반화한다. | 재설정을 처음부터 다시 시작한다. |
| 멱등·auth-flow 복구 기간 만료 | grant를 새로 발급하지 않는다. | 새 Challenge로 다시 검증한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `VerifyChallengeHandler` |
| Aggregate / Entity | `PasswordReset`, `VerificationChallenge` |
| Repository / Read Model | PasswordReset·Challenge Repository, `IdempotencyRecord` |
| Port / Adapter | PasswordReset grant issuer, SecretGeneratorPort |
| Domain / Integration Event | Challenge 검증·비밀번호 재설정 검증 감사 OutboxEvent |

## 관측성과 운영

- 로그에는 challenge ID, purpose, channel과 일반화된 결과만 남긴다.
- `auth_challenge_verified_total`, 실패·만료·rate-limit 비율을 관측한다.
- 검증, row lock, grant hash 교체와 outbox 저장을 별도 span으로 기록한다.
- code, reset grant와 전체 요청·응답 body는 sampling하지 않는다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 웹과 모바일 `oneOf` 응답 및 `credentialDelivery`가 일치한다.
- code가 공용 `VerificationCode` schema를 통과해야 한다.
- 같은 실패 재시도가 attempt count를 중복 증가시키지 않는다.
- 실패 뒤 변경한 code를 기존 key로 제출하면 멱등 충돌이 발생하고 새 key에서는 새 검증 시도로 처리된다.
- 동시 성공 요청 중 하나만 Challenge와 reset grant를 소비한다.
- 모바일 성공 응답 유실 후 같은 key에서 새 grant를 발급하고 이전 grant가 즉시 무효인지 검증한다.
- 민감값이 로그, trace, metric label과 IdempotencyRecord에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.310-01 비밀번호 재설정](../A_300_50-sequence/SCN_A_310_01_password_reset.md)
- 관련 API: `API.A.300-10`, `API.A.300-11`, `API.A.300-13`
- 여러 참여자의 Mermaid 다이어그램은 `A_300_50-sequence` 문서에서 관리한다.

## 호환성과 변경 정책

- 공개 가능한 상태 필드 추가는 기존 discriminator 의미를 유지할 때 하위 호환으로 처리한다.
- email link token 등 새 proof 형식은 새 enum과 schema variant를 추가한 뒤 제공한다.
- reset grant 전달 방식 변경은 새 API 버전으로 처리한다.

## 확인 필요

- 모바일 reset grant 재발급을 허용할 auth-flow·멱등 복구 기간의 초기 운영값을 확정해야 한다.
