---
id: API.A.300-19
title: 인증 수단 연동 Challenge 발급 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-19 인증 수단 연동 Challenge 발급

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/method-links/{linkIntentId}/challenges` |
| operationId | `issueMethodLinkChallenge` |
| 역할 | requested phone Link에 `sms` 소유 확인 Challenge를 발급한다. |
| API 유형 | Command |
| 인증 | 웹·모바일 공통 Bearer access JWT |
| 권한 | requested Link의 `user_id`와 현재 Session 사용자 일치 |
| 노출 범위 | public, phone MVP |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_19_issue_method_link_challenge.yaml](openapi/paths/API_A_300_19_issue_method_link_challenge.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

이번 버전의 `channel`은 `sms`로 제한한다. request/response schema, 응답 header, HTTP 상태, 오류 body와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [인증 시퀀스 인덱스](../A_300_50-sequence/README.md) |

## 책임과 경계

- 현재 사용자가 소유한 requested phone Link에 `identity_link` 목적 SMS Challenge를 만든다.
- Challenge와 delivery outbox를 원자적으로 저장하고 relay가 가상 SMS Adapter를 호출하게 한다.
- requested Link가 없거나 만료되면 Challenge를 만들지 않는다.
- Link 활성화와 계정 병합은 이 Endpoint에서 수행하지 않는다.

## 보안과 개인정보

- Istio가 access JWT와 active Session을 검증한다. Bearer 보호 API에는 CSRF 검사를 적용하지 않는다.
- 휴대폰 번호와 code 원문을 응답, 로그, trace, metric label과 event에 남기지 않는다.
- 응답의 마스킹 destination은 현재 사용자가 직접 시작한 Link 범위에서만 제공한다.
- Link 부재와 다른 사용자 소유를 같은 not-found 공개 오류로 일반화한다.

## 처리 규칙

1. 현재 Session과 requested Link의 user·상태·만료 binding을 확인한다.
2. `channel=sms`와 VerificationPolicy의 재발송 제한을 검증한다.
3. `context_id=linkIntentId`, `purpose=identity_link`인 Challenge를 만든다.
4. Challenge와 delivery outbox를 한 트랜잭션에 저장한다.
5. 마스킹 destination과 Challenge metadata를 반환하고 실제 발송은 relay에 맡긴다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `SessionStatusProjection` | 사용 | 인증 수단을 연결하려는 사용자의 Session 상태를 먼저 확인한다. |
| `AuthenticationPolicySnapshotProjection` | 사용 | challenge 수명, 재발급 간격과 발송 제한을 활성 정책 version으로 적용한다. |
| 요청된 `IdentityLink`, `VerificationChallenge`, `IdempotencyRecord`, `OutboxEvent` | 우회 | 연결 요청 소유권, 기존 challenge 교체와 발송 요청을 PostgreSQL 트랜잭션으로 확정한다. |
| challenge code와 발송 secret | 사용하지 않음 | 일회용 비밀값을 Redis projection으로 복제하지 않는다. |

## 상태 변경과 트랜잭션

- 시작 상태는 만료되지 않은 `IdentityLink.requested`다.
- Challenge, Link의 proof challenge 참조, delivery payload와 OutboxEvent를 한 트랜잭션에 저장한다.
- 같은 context의 기존 issued Challenge를 교체하면 기존 항목을 `revoked`로 닫는다.
- 성공 뒤 Link는 계속 `requested`이며 소유 검증 전에는 active가 되지 않는다.
- SMS Adapter 호출은 commit 이후 relay가 수행한다.

## 멱등성과 동시성

- 멱등 범위는 API ID, link intent, 현재 사용자와 `Idempotency-Key`다.
- request fingerprint에 channel과 Link reference를 포함하고 민감값은 keyed HMAC으로만 저장한다.
- 같은 key와 같은 요청은 같은 Challenge metadata를 반환하고 SMS를 다시 보내지 않는다.
- 같은 key와 다른 요청은 충돌로 거부한다.
- context·purpose별 active Challenge 유일성과 Link version으로 동시 발급을 제어한다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ErrorResponse schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| requested Link 부재·소유 불일치 | Link 존재와 사용자 정보를 구분하지 않는다. | 연동 시작 상태를 확인한다. |
| requested Link 만료 | 대상 Identity 정보를 공개하지 않는다. | 재인증 후 새 연동을 시작한다. |
| 발급 제한 초과 | 내부 제한값을 숨긴다. | `Retry-After` 뒤 새 key로 요청한다. |
| delivery 장애 | 소유 확인 성공으로 처리하지 않는다. | 발송 상태 정책에 따라 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `RequestVerificationChallengeHandler` |
| Aggregate / Entity | `IdentityLink`, `VerificationChallenge`, `Session` |
| Repository / Read Model | IdentityLink·Challenge Repository, `IdempotencyRecord` |
| Port / Adapter | `SmsVerificationSender`, Outbox Relay |
| Domain / Integration Event | `VerificationDeliveryRequested`, 연동 Challenge 감사 OutboxEvent |

## 관측성과 운영

- 로그에는 link intent ID, challenge ID, channel과 일반화된 결과만 남긴다.
- `auth_challenge_issued_total`, `auth_delivery_total`, rate-limit과 outbox backlog를 관측한다.
- Link 조회, Challenge 저장, relay와 SMS Adapter를 별도 span으로 기록한다.
- 전화번호, code와 delivery payload는 sampling하지 않는다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- `channel=sms` 이외의 입력을 거부한다.
- 다른 사용자의 Link와 없는 Link가 같은 공개 오류로 처리된다.
- 만료된 Link에는 Challenge와 outbox를 만들지 않는다.
- 같은 key 재요청과 동시 요청이 Challenge나 발송을 중복 만들지 않는다.
- 휴대폰 번호와 code가 로그, trace, metric label, event에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [인증 처리 시퀀스 인덱스](../A_300_50-sequence/README.md)
- 관련 API: `API.A.300-17`, `API.A.300-18`, `API.A.300-20`
- 인증 수단 연동 전용 시퀀스는 추가 시 `A_300_50-sequence`에서 관리한다.

## 호환성과 변경 정책

- Challenge metadata의 비민감 필드 추가는 하위 호환으로 처리한다.
- email·provider proof 채널은 별도 schema variant와 검증 방식을 함께 추가한다.
- Link 부재·소유 불일치의 비공개 원칙은 버전과 관계없이 유지한다.

## 확인 필요

- 마스킹 destination 표기 형식과 locale별 규칙을 확정해야 한다.
