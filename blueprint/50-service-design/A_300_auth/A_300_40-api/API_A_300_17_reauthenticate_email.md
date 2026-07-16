---
id: API.A.300-17
title: 이메일 재인증 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-17 이메일 재인증

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/reauthentications/email` |
| operationId | `reauthenticateEmail` |
| 역할 | 현재 비밀번호를 다시 확인하고 목적 한정 ReauthenticationProof와 회전된 Session credential을 발급한다. |
| API 유형 | Command |
| 인증 | 웹·모바일 공통 Bearer access JWT와 현재 비밀번호 |
| 권한 | active 이메일 Link, active PasswordCredential와 active UserAuthState |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_17_reauthenticate_email.yaml](openapi/paths/API_A_300_17_reauthenticate_email.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

웹·모바일 credential 회전 응답의 `oneOf`, request schema, 응답 header, HTTP 상태, 오류 body와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [SCN.A.300-03](../A_300_50-sequence/SCN_A_300_03_phone_replacement.md) |

## 책임과 경계

- 현재 비밀번호와 active 이메일 IdentityLink를 검증해 고위험 작업용 단기 proof를 만든다.
- Session 인증 근거를 이메일 비밀번호로 재바인딩하고 채널별 credential을 회전한다.
- proof는 요청한 목적 하나에만 사용할 수 있으며 role·permission을 부여하지 않는다.
- 휴대폰 교체나 인증 수단 연동 자체는 후속 Endpoint에서 처리한다.

## 보안과 개인정보

- Istio가 access JWT와 active Session을 검증하고 Auth는 현재 비밀번호를 추가로 확인한다.
- 비밀번호, proof, cookie, access/refresh token 원문을 저장·로그·trace·event에 남기지 않는다.
- 공개 목적은 `replace_phone`, `link_identity`, `seller_order_export`, `seller_member_manage`, `seller_account_update`, `seller_partnership_respond`로 제한하고 `manual_recovery`는 운영 API 경계에 둔다.
- seller purpose proof는 seller ID와 실제 권한을 담지 않는다. 소비 workload가 Session, target seller와 operation을 binding하고 Context 판매자가 현재 membership·permission을 별도로 검증한다.
- 웹은 새 access JWT와 회전된 refresh cookie를, 모바일은 새 access/refresh token을 같은 성공 응답에서 전달한다.
- 응답 유실 복구는 같은 Session·operation·key·request fingerprint의 `rotated_pending_delivery`에만 이전 credential hash를 허용하고 최초 credential·proof 응답을 short-TTL 암호문에서 재생한다.

## 처리 규칙

1. 현재 Session, UserAuthState와 요청 purpose allowlist를 확인한다.
2. 같은 사용자의 active 이메일 Link와 active PasswordCredential로 현재 비밀번호를 검증한다.
3. Session의 인증 근거와 `last_authenticated_at`을 이메일 비밀번호 기준으로 갱신한다.
4. 기존 SessionCredential을 회전하고 목적 한정 ReauthenticationProof를 만든다.
5. 웹 또는 모바일 채널에 맞는 새 credential과 proof를 클라이언트에 반환한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `SessionStatusProjection` | 사용·갱신 | 요청자의 Session을 먼저 확인하고, 재인증 뒤 Session credential을 회전하거나 version을 변경하면 커밋 후 projection도 갱신한다. |
| `AuthenticationPolicySnapshotProjection` | 사용 | 재인증 유효 시간과 credential 정책을 활성 정책 version으로 검사한다. |
| `PasswordCredential`, `SessionCredential`, `ReauthenticationProof`, `IdempotencyReplayPayload` | 우회 | 비밀번호 검증, proof 발급, credential 회전과 재사용 방지는 PostgreSQL 원장과 트랜잭션이 필요하다. |

## 상태 변경과 트랜잭션

- 시작 상태는 active Session과 active SessionCredential이다.
- 비밀번호 검증 성공 뒤 Session 재바인딩, credential 회전, proof 저장과 감사 OutboxEvent를 한 트랜잭션에 저장한다.
- 같은 트랜잭션에 `rotated_pending_delivery`, completed IdempotencyRecord와 최초 credential·proof 응답의 short-TTL replay ciphertext를 저장한다.
- 두 채널 모두 refresh credential은 같은 family에서 회전하고 새 access JWT의 `jti`가 바뀐다.
- 실패 시 Session credential과 proof를 변경하지 않으며 정책상 로그인 실패 기록만 원자적으로 갱신한다.
- proof 만료 시각은 Session absolute 만료 시각을 넘지 않는다.

## 멱등성과 동시성

- 멱등 범위는 API ID, Session, purpose와 `Idempotency-Key`다.
- 비밀번호 원문 대신 canonical request의 keyed HMAC fingerprint만 저장한다.
- 같은 key와 같은 실패는 실패 횟수를 다시 늘리지 않고 같은 오류를 재생한다.
- 같은 key와 같은 성공은 기존 proof와 암호화한 채널별 credential 결과를 복구한다.
- 회전 전 credential hash는 네 binding이 모두 맞는 전용 복구 분기에서만 허용하며 일반 인증과 다른 API에는 사용할 수 없다.
- SessionCredential row lock과 proof의 단일 활성 결과로 중복 회전을 막는다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ErrorResponse schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| 현재 비밀번호 불일치 | 계정·credential 상태를 구분하지 않는다. | 입력을 확인하고 제한 정책 안에서 재시도한다. |
| Session 없음·폐기 | 이메일 Link 존재 여부를 공개하지 않는다. | 다시 로그인한다. |
| 사용자 제한 | 내부 제한 사유를 숨긴다. | 지원 절차를 안내한다. |
| 성공 응답 복구 TTL 만료 | proof나 token을 새로 재생하지 않는다. | 새 재인증 요청을 시작한다. |
| 복구 binding 불일치 | 이전 credential의 존재와 상태를 더 공개하지 않는다. | 현재 유효한 Session으로 새 재인증을 시작한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `ReauthenticateWithEmailHandler` |
| Aggregate / Entity | `Session`, `SessionCredential`, `PasswordCredential`, `IdentityLink`, `ReauthenticationProof` |
| Repository / Read Model | Session·Credential·Proof Repository, `IdempotencyRecord`와 replay store |
| Port / Adapter | PasswordHasherPort, access JWT signer |
| Domain / Integration Event | 재인증 성공·실패와 credential 회전 감사 OutboxEvent |

## 관측성과 운영

- 로그에는 session ID, purpose, channel과 일반화된 결과만 남긴다.
- 재인증 성공·실패·제한, credential 회전과 latency를 관측한다.
- password verifier, Session lock, proof 저장과 signer를 별도 span으로 기록한다.
- 비밀번호, proof, token과 전체 body를 sampling하지 않는다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 웹 성공에서 새 access JWT와 refresh cookie가 함께 전달된다.
- 모바일 성공에서 access/refresh token이 회전되고 이전 refresh token이 비활성화된다.
- purpose가 proof에 고정되고 다른 목적에서 소비되지 않는다.
- seller purpose가 다른 seller, Session, operation에서 소비되지 않고 membership 변경 뒤 권한 근거로 재사용되지 않는다.
- 같은 key 재시도가 credential과 proof를 중복 만들지 않는다.
- 회전 전 credential은 정확한 복구 binding에서만 성공하고 일반 보호 API에서는 거부된다.
- 비밀번호와 credential 원문이 로그, trace, event와 IdempotencyRecord에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.300-03 휴대폰 번호 교체](../A_300_50-sequence/SCN_A_300_03_phone_replacement.md)
- 관련 API: `API.A.300-18`, `API.A.300-21`, `API.A.300-22`, `API.A.300-23`
- 판매자 소비 API: [API.A.200 operation catalog](../../A_200_seller/A_200_40-api/operation-catalog.md)
- 여러 참여자의 Mermaid 다이어그램은 `A_300_50-sequence` 문서에서 관리한다.

## 호환성과 변경 정책

- 새 public purpose 추가는 소비 Endpoint와 함께 하위 호환 enum 확장으로 제공한다.
- proof 형식과 credential 회전 의미 변경은 새 API 버전으로 제공한다.
- 운영 전용 purpose는 public allowlist에 추가하지 않는다.

## 확인 필요

- 재인증 credential·proof 전달 복구 TTL의 초기값을 확정해야 한다.
