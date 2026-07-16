---
id: API.A.300-20
title: 인증 수단 연동 완료 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-20 인증 수단 연동 완료

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/method-links/{linkIntentId}/complete` |
| operationId | `completeMethodLink` |
| 역할 | SMS code proof를 소비하고 requested phone IdentityLink를 active로 전환한다. |
| API 유형 | Command |
| 인증 | 웹·모바일 공통 Bearer access JWT |
| 권한 | requested Link 사용자, 현재 Session과 Challenge context binding 일치 |
| 노출 범위 | public, phone MVP |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_20_complete_method_link.yaml](openapi/paths/API_A_300_20_complete_method_link.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

이번 버전은 phone Link와 code proof만 지원한다. request/response schema, 응답 header, HTTP 상태, 오류 body와 wire 예시는 OpenAPI를 기준으로 한다.

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

- requested phone Link와 `identity_link` SMS Challenge의 소유·context binding을 검증한다.
- Challenge 검증·소비와 IdentityLink 활성화를 한 로컬 트랜잭션으로 처리한다.
- 다른 `user_id`에 active 연결된 Identity를 이전하거나 두 계정을 병합하지 않는다.
- 기존 active Link 해제나 휴대폰 번호 교체는 별도 API 책임이다.

## 보안과 개인정보

- Istio가 access JWT와 active Session을 검증한다. Bearer 보호 API에는 CSRF 검사를 적용하지 않는다.
- code, 전화번호와 기존 Link의 `user_id`를 응답·로그·trace·metric label에 남기지 않는다.
- Link 부재와 다른 사용자 소유는 같은 not-found 오류로 일반화한다.
- active Link 충돌은 기존 사용자나 연결 이력을 노출하지 않는 공개 code로 변환한다.

## 처리 규칙

1. 현재 Session과 requested Link의 user·status·만료를 검증한다.
2. Challenge ID, `identity_link` 목적, Link context와 code proof를 검증한다.
3. Identity가 다른 사용자에게 active 연결되지 않았는지 DB 제약과 함께 확인한다.
4. Challenge를 소비하고 Identity owner와 requested Link를 active로 전환한다.
5. 활성화된 Link의 안전한 식별자와 method만 반환한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `SessionStatusProjection` | 사용 | 완료 요청을 보낸 사용자의 Session 상태를 먼저 확인한다. 인증 수단이 추가되어도 Session 상태나 version은 바뀌지 않으므로 projection을 갱신하지 않는다. |
| `AuthenticationPolicySnapshotProjection` | 사용 | challenge 검증 횟수와 연결 완료 조건을 활성 정책 version으로 검사한다. |
| `Identity`, `IdentityLink`, `VerificationChallenge`, `IdempotencyRecord` | 우회 | challenge 일회성 소비와 Identity 연결을 PostgreSQL의 unique 제약과 트랜잭션으로 확정한다. |

## 상태 변경과 트랜잭션

- 시작 상태는 `IdentityLink.requested`와 `VerificationChallenge.issued`다.
- Challenge 검증·소비, Identity owner 확정, Link `active`, 감사 OutboxEvent를 한 트랜잭션에 저장한다.
- 성공 종료 상태는 `IdentityLink.active`와 `VerificationChallenge.verified`다.
- 검증·유일성 확인 중 실패하면 Challenge와 Link 상태를 부분 변경하지 않는다.
- 외부 Provider 호출 없이 로컬 transaction으로 완료한다.

## 멱등성과 동시성

- 멱등 범위는 API ID, link intent, 현재 사용자와 `Idempotency-Key`다.
- code 원문 대신 canonical request의 keyed HMAC fingerprint만 저장한다.
- 같은 key는 전송 결과가 불확실한 동일 request body의 재전송에만 사용하며, 기존 active Link 결과를 재생하고 Challenge를 다시 소비하지 않는다.
- 검증 실패 뒤 새로 입력하거나 수정한 code는 새 `Idempotency-Key`로 제출한다. 기존 key와 다른 body는 멱등 충돌로 거부한다.
- 같은 key와 다른 요청은 충돌로 거부한다.
- Challenge·Link row lock과 Identity별 active Link unique constraint로 동시 완료를 제어한다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ErrorResponse schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| Link 부재·소유 불일치 | Link와 사용자 존재를 구분하지 않는다. | 연동 시작 상태를 다시 확인한다. |
| Link 또는 Challenge 만료 | Identity 정보를 공개하지 않는다. | 재인증 후 새 연동을 시작한다. |
| code 검증 실패 | 남은 내부 횟수를 기본적으로 숨긴다. | 전송 불확실성은 같은 key·같은 body로 재시도하고, 새로 입력하거나 수정한 code는 새 key로 제출한다. |
| 다른 사용자와 active Link 충돌 | 기존 `user_id`를 숨긴다. | 자동 병합 없이 지원 절차를 안내한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `CompleteIdentityLinkHandler`, `VerifyChallengeHandler` |
| Aggregate / Entity | `Identity`, `IdentityLink`, `VerificationChallenge`, `Session` |
| Repository / Read Model | Identity·Link·Challenge Repository, `IdempotencyRecord` |
| Port / Adapter | 해당 없음 |
| Domain / Integration Event | 인증 수단 연동 성공·실패 감사 OutboxEvent |

## 관측성과 운영

- 로그에는 link intent ID, challenge ID, method와 일반화된 결과만 남긴다.
- 연동 완료 성공·검증 실패·충돌·만료와 transaction latency를 관측한다.
- Challenge lock, Identity 유일성, Link 활성화와 outbox 저장을 span으로 기록한다.
- code와 전화번호, 전체 요청 body는 sampling하지 않는다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- phone Link와 code proof 이외의 입력을 거부한다.
- Link와 Challenge context·소유 binding이 모두 검증된다.
- transaction 실패 시 Challenge와 Link가 기존 상태를 유지한다.
- 같은 key·동시 요청이 Link 활성화와 감사 event를 중복 만들지 않는다.
- 실패 뒤 변경한 code를 기존 key로 제출하면 멱등 충돌이 발생하고 새 key에서는 새 검증 시도로 처리된다.
- 다른 사용자에게 연결된 Identity의 소유 정보를 공개하지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [인증 처리 시퀀스 인덱스](../A_300_50-sequence/README.md)
- 관련 API: `API.A.300-17`, `API.A.300-18`, `API.A.300-19`
- 인증 수단 연동 전용 시퀀스는 추가 시 `A_300_50-sequence`에서 관리한다.

## 호환성과 변경 정책

- Link 결과의 비민감 metadata 추가는 하위 호환으로 처리한다.
- email·provider proof는 별도 request variant와 상태 전이 규칙을 함께 추가한다.
- 계정 병합과 기존 active Link의 암묵적 이전은 어떤 하위 버전에도 추가하지 않는다.

## 확인 필요

- 인증 수단 연동 성공 event를 외부 Integration Event로 승격할지 Audit 전송 범위에서 확정한다.
