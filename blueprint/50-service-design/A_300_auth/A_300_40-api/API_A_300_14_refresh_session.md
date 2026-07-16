---
id: API.A.300-14
title: Session 갱신 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-14 Session 갱신

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/sessions/refresh` |
| operationId | `refreshSession` |
| 역할 | 채널별 refresh token을 회전하고 새 access JWT를 발급한다. |
| API 유형 | Command |
| 인증 | 웹 `__Host-dm_refresh` cookie·CSRF·Origin 또는 모바일 `X-Refresh-Token` header |
| 권한 | active Session, active refresh family와 active UserAuthState |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_14_refresh_session.yaml](openapi/paths/API_A_300_14_refresh_session.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

웹과 모바일의 credential 위치는 OpenAPI security alternative로 구분한다. request body에는 refresh token을 넣지 않으며 응답 schema, header, HTTP 상태와 wire 예시는 Path Item을 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [SCN.A.300-02](../A_300_50-sequence/SCN_A_300_02_phone_signin_refresh.md) |

## 책임과 경계

- refresh credential을 검증하고 같은 Session의 credential을 회전한다.
- UserAuthState가 active인지 확인하고 새 access JWT에는 인증 allowlist claim만 넣는다.
- 웹은 새 access JWT를 본문으로 받고 회전된 refresh token은 HttpOnly cookie로 받는다.
- 사용자 프로필과 업무 리소스 권한은 조회하거나 token에 포함하지 않는다.

## 보안과 개인정보

- refresh token은 opaque secret이며 cookie/header 외의 request body, 로그, trace, event와 IdempotencyRecord에 남기지 않는다.
- access JWT에는 `iss`, `sub`, `sid`, `aud`, `iat`, `exp`, `jti`만 넣는다.
- rotated 또는 revoked token의 다른-key 사용은 재시도가 아니라 reuse 탐지로 처리한다.
- 성공 응답과 암호화 replay payload는 `no-store`이며 짧은 복구 TTL을 적용한다.

## 처리 규칙

1. token lookup digest로 SessionCredential과 Session을 찾고 기본 상태를 확인한다.
2. 최신 UserAuthState를 조회하며 상태를 확정할 수 없으면 fail closed로 처리한다.
3. credential row lock을 획득하고 이전 credential을 `rotated`로 바꾼다.
4. 같은 refresh family에 새 credential을 만들고 allowlist claim의 access JWT를 발급한다.
5. 웹은 access JWT와 refresh cookie를, 모바일은 access JWT와 refresh token을 반환하고 암호화 replay reference를 저장한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `Session`, `SessionCredential`, `IdempotencyRecord`, `IdempotencyReplayPayload` | 우회 | refresh token 회전과 재사용 탐지는 같은 credential row를 잠근 상태에서 PostgreSQL 원장으로 판정해야 한다. |
| `AuthenticationPolicySnapshotProjection` | 사용 | access·refresh 수명과 회전 정책을 활성 정책 version 기준으로 적용한다. |
| `SessionStatusProjection` | 갱신·무효화 | 정상 회전 뒤에는 Session version과 만료 시각을 갱신하고, 재사용이 탐지되면 `revoked` 상태로 즉시 무효화한다. |

## 상태 변경과 트랜잭션

- 시작 상태는 active Session과 active refresh credential이다.
- credential row lock, 이전 credential 회전, 새 credential과 감사 OutboxEvent를 한 트랜잭션에 저장한다.
- reuse가 탐지되면 Session과 refresh family를 폐기하고 보안 감사 event를 같은 트랜잭션에 저장한다.
- 새 access JWT는 저장 Entity가 아니며 commit된 결과에 대해서만 생성·전달한다.

## 멱등성과 동시성

- 멱등 범위는 API ID, refresh credential scope와 `Idempotency-Key`다.
- token 원문 대신 keyed digest와 canonical request HMAC fingerprint만 저장한다.
- 같은 token과 같은 key는 복구 TTL 동안 암호화한 최초 회전 결과를 재생한다.
- 같은 이전 token과 다른 key는 reuse로 판단해 Session과 family를 폐기한다.
- credential row lock으로 동시에 들어온 회전 요청 중 하나만 성공시킨다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ErrorResponse schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| Session 폐기 또는 token 재사용 | token 상태의 세부 정보를 숨긴다. | 로컬 token을 삭제하고 다시 로그인한다. |
| 사용자 제한 | 제한 세부 사유를 노출하지 않는다. | 지원 절차를 안내하고 재발급하지 않는다. |
| 같은-key replay TTL 만료 | 이전 token이나 응답을 다시 발급하지 않는다. | 다시 로그인한다. |
| 사용자 인증 상태·저장소 확인 장애 | active로 추측해 새 token을 만들지 않는다. | `Retry-After` 정책에 따라 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `RefreshSessionHandler` |
| Aggregate / Entity | `Session`, `SessionCredential`, `UserAuthState` |
| Repository / Read Model | Session·Credential Repository, `IdempotencyRecord`와 replay payload store |
| Port / Adapter | access JWT signer |
| Domain / Integration Event | Session refresh·reuse 탐지 감사 OutboxEvent |

## 관측성과 운영

- 로그에는 session ID, refresh family ID와 일반화된 결과만 남기고 token은 남기지 않는다.
- `auth_session_refresh_total`, `auth_refresh_reuse_detected_total`, refresh latency를 관측한다.
- credential lock, transaction과 token signer를 별도 span으로 기록한다.
- refresh spike와 reuse 탐지는 보안 경보 대상으로 운영한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 정상 회전에서 이전 credential이 비활성화되고 새 credential 하나만 active가 된다.
- 같은 key 응답 유실 재시도에서 채널별 동일 논리 결과를 복구한다.
- 같은 이전 token의 다른-key 사용이 family 폐기로 이어진다.
- 동시 refresh 중 하나만 회전에 성공한다.
- token 원문과 개인정보가 저장소 메타데이터, 로그, trace, event에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.300-02 휴대폰 로그인과 refresh token 회전](../A_300_50-sequence/SCN_A_300_02_phone_signin_refresh.md), [SCN.A.300-05 웹 JWT 인증](../A_300_50-sequence/SCN_A_300_05_web_jwt_authentication.md)
- 관련 API: `API.A.300-08`, `API.A.300-09`
- 여러 참여자의 Mermaid 다이어그램은 `A_300_50-sequence` 문서에서 관리한다.

## 호환성과 변경 정책

- token 만료 시각과 공개 가능한 Session metadata 추가는 하위 호환으로 처리한다.
- refresh credential의 전달 위치와 rotation 의미 변경은 새 API 버전으로 제공한다.
- JWT claim 변경은 issuer/audience consumer의 호환성을 고려해 별도로 versioning한다.

## 확인 필요

- 암호화 refresh replay TTL과 family 단위 폐기 보존 기간의 초기 운영값을 확정해야 한다.
