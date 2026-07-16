---
id: API.A.300-18
title: 인증 수단 연동 시작 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-18 인증 수단 연동 시작

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/method-links` |
| operationId | `startMethodLink` |
| 역할 | 새 휴대폰 인증 식별자의 requested Link를 만들거나 같은 사용자의 기존 active Link를 멱등 성공으로 반환한다. |
| API 유형 | Command |
| 인증 | 웹·모바일 공통 Bearer access JWT와 `link_identity` 목적 proof |
| 권한 | 현재 Session 사용자와 proof의 user·session·purpose binding 일치 |
| 노출 범위 | public, phone MVP |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_18_start_method_link.yaml](openapi/paths/API_A_300_18_start_method_link.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

이번 버전의 `method`는 `phone`으로 제한한다. request/response schema, 응답 header, HTTP 상태, 오류 body와 wire 예시는 OpenAPI를 기준으로 한다.

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

- `link_identity` 목적 proof를 소비하고 신규 phone IdentityLink를 requested 상태로 만들거나 같은 사용자의 기존 active Link를 반환한다.
- `linkIntentId`는 별도 Aggregate가 아니라 requested 상태의 IdentityLink ID이며, 기존 active 응답은 `identityLinkId`를 사용한다.
- 이 단계에서는 SMS Challenge를 발급하거나 메시지를 보내지 않는다.
- 계정 병합과 이미 다른 `user_id`에 연결된 Identity의 이전을 수행하지 않는다.

## 보안과 개인정보

- Istio가 access JWT와 active Session을 검증한다. Bearer 보호 API에는 CSRF 검사를 적용하지 않는다.
- 휴대폰 번호 원문과 proof를 로그, trace, metric label, event와 IdempotencyRecord에 남기지 않는다.
- Identity 충돌 응답에는 기존 `user_id`나 연결 상태의 세부 정보를 포함하지 않는다.
- proof는 user, Session, `link_identity`, 만료와 단일 소비 조건에 바인딩한다.

## 처리 규칙

1. Session, UserAuthState와 ReauthenticationProof의 목적·소유·만료를 확인한다.
2. 휴대폰 번호를 canonical form으로 정규화하고 Identity lookup key를 계산한다.
3. 같은 사용자에게 이미 active 연결된 Identity면 proof를 소비하고 기존 Link를 `200 active`로 반환한다.
4. 다른 사용자에게 active 연결된 Identity는 계정 병합 없이 거부한다.
5. 신규 대상이면 proof 소비와 phone Identity·requested IdentityLink 생성을 원자적으로 처리하고 `201 requested`를 반환한다.
6. requested 분기의 Challenge 발급은 후속 API에 맡긴다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `SessionStatusProjection` | 사용 | 연결 작업을 시작한 사용자의 Session이 아직 유효한지 빠르게 확인한다. 이 API는 Session 상태나 version을 바꾸지 않으므로 projection을 갱신하지 않는다. |
| `AuthenticationPolicySnapshotProjection` | 사용 | reauth proof 수명과 인증 수단 연결 정책을 활성 정책 version으로 검사한다. |
| `ReauthenticationProof`, `Identity`, `IdentityLink`, `IdempotencyRecord` | 우회 | proof 소비와 중복 Identity 연결 검사는 PostgreSQL의 unique 제약과 트랜잭션으로 확정한다. |

## 상태 변경과 트랜잭션

- 시작 상태는 active Session과 미소비 `link_identity` ReauthenticationProof다.
- proof row lock·소비와 감사 OutboxEvent를 공통 트랜잭션에 저장한다.
- 기존 active 분기는 새 Identity나 Link를 만들지 않고 `200`을 반환한다.
- 신규 분기는 pending Identity와 requested IdentityLink를 만들고 `201`을 반환한다.
- 저장 실패 시 proof를 소비하거나 미완료 Link를 남기지 않는다.
- 성공 종료 상태는 기존 `IdentityLink.active` 또는 `intent_expires_at`을 가진 신규 `IdentityLink.requested`다.
- 외부 SMS Adapter는 호출하지 않는다.

## 멱등성과 동시성

- 멱등 범위는 API ID, Session 사용자와 `Idempotency-Key`다.
- 전화번호와 proof 원문 대신 canonical request의 keyed HMAC fingerprint만 저장한다.
- 같은 key와 같은 요청은 최초 결과인 `200 active` 또는 `201 requested`를 재생하고 proof를 다시 소비하지 않는다.
- 같은 key와 다른 요청은 충돌로 거부한다.
- Identity lookup unique constraint, proof row lock과 requested Link 유일성으로 경합을 제어한다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ErrorResponse schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| Session 또는 recent auth 부족 | Identity 존재 여부를 공개하지 않는다. | 로그인 또는 이메일 재인증을 수행한다. |
| proof 만료·소비·목적 불일치 | 세부 binding을 숨긴다. | `link_identity` 목적 재인증을 다시 수행한다. |
| 같은 사용자의 기존 active Link | 소유자 상세 없이 안전한 Link ID와 method만 반환한다. | `200 active`를 성공으로 처리한다. |
| 다른 사용자와 active Link 충돌 | 기존 사용자 식별자를 숨긴다. | 자동 병합 없이 지원 절차를 안내한다. |
| transaction 실패 | proof와 Link를 부분 저장하지 않는다. | 같은 key로 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `StartIdentityLinkHandler` |
| Aggregate / Entity | `Identity`, `IdentityLink`, `ReauthenticationProof`, `Session` |
| Repository / Read Model | Identity·Link·Proof Repository, `IdempotencyRecord` |
| Port / Adapter | PhoneNormalizer, keyed lookup service |
| Domain / Integration Event | 인증 수단 연동 요청 감사 OutboxEvent |

## 관측성과 운영

- 로그에는 link intent ID, method, purpose와 일반화된 결과만 남긴다.
- 연동 시작 성공·충돌·proof 만료와 transaction latency를 관측한다.
- proof lock, Identity lookup, Link 저장과 outbox 저장을 별도 span으로 기록한다.
- 휴대폰 번호, proof와 전체 body는 sampling하지 않는다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- `method=phone` 이외의 입력을 거부한다.
- proof 목적·사용자·Session·1회성이 모두 검증된다.
- 같은 사용자의 기존 active Link는 새 Link를 만들지 않고 `200 active`로 반환한다.
- 신규 대상은 `201 requested`로 반환하고 후속 Challenge 발급 전까지 active로 취급하지 않는다.
- 저장 실패 시 proof와 requested Link가 모두 원상태를 유지한다.
- 같은 key와 동시 요청이 Link를 중복 만들지 않는다.
- 다른 사용자와 충돌한 Identity의 소유 정보를 공개하지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [인증 처리 시퀀스 인덱스](../A_300_50-sequence/README.md)
- 관련 API: `API.A.300-17`, `API.A.300-19`, `API.A.300-20`
- 인증 수단 연동 전용 시퀀스는 추가 시 `A_300_50-sequence`에서 관리한다.

## 호환성과 변경 정책

- `200 active`와 `201 requested`의 상태 의미를 유지하는 비민감 metadata 추가는 하위 호환으로 처리한다.
- email·provider 연동은 method별 request variant와 proof 검증 방식을 함께 추가한다.
- 계정 병합을 이 API의 하위 호환 기능으로 추가하지 않는다.

## 확인 필요

- 인증 수단 연동 전용 시퀀스 문서의 Scenario ID를 확정해야 한다.
