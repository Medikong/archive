---
id: API.A.300-14
title: 모바일 Session 갱신 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-10
---

# API.A.300-14 모바일 Session 갱신

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/sessions/refresh` |
| operationId | `refreshSession` |
| 역할 | 모바일 refresh token을 회전하고 최신 권한의 access JWT를 발급한다. |
| API 유형 | Command |
| 인증 | request body의 opaque refresh credential |
| 권한 | active Session, active refresh family와 active UserAuthState |
| 노출 범위 | public, mobile only |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_14_refresh_session.yaml](openapi/paths/API_A_300_14_refresh_session.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

Refresh credential은 body에 있으므로 operation의 OpenAPI security는 빈 배열로 선언한다. request/response schema, 응답 header, HTTP 상태, 오류 body와 wire 예시는 Path Item을 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [SCN.A.300-02](../../../80-sequence/A_300_auth/SCN_A_300_02_phone_signin_refresh.md) |

## 책임과 경계

- refresh credential을 검증하고 같은 Session의 credential을 회전한다.
- Authorization Source에서 최신 AccessGrant를 얻어 새 access JWT에 반영한다.
- 웹 server Session은 이 Endpoint를 사용하지 않는다.
- 사용자 프로필과 업무 리소스 권한은 조회하거나 token에 포함하지 않는다.

## 보안과 개인정보

- refresh token은 opaque secret이며 request body 외의 로그, trace, event와 IdempotencyRecord에 남기지 않는다.
- access JWT에는 `user_id`, Session과 최소 role/version claim만 넣고 이메일·휴대폰 번호를 제외한다.
- rotated 또는 revoked token의 다른-key 사용은 재시도가 아니라 reuse 탐지로 처리한다.
- 성공 응답과 암호화 replay payload는 `no-store`이며 짧은 복구 TTL을 적용한다.

## 처리 규칙

1. token lookup digest로 SessionCredential과 Session을 찾고 기본 상태를 확인한다.
2. 최신 UserAuthState와 AccessGrant를 조회하며 필수 원천 장애 시 fail closed로 처리한다.
3. credential row lock을 획득하고 이전 credential을 `rotated`로 바꾼다.
4. 같은 refresh family에 새 credential을 만들고 최신 claim의 access JWT를 발급한다.
5. 새 token 묶음을 클라이언트에 직접 반환하고 암호화 replay reference를 저장한다.

## 상태 변경과 트랜잭션

- 시작 상태는 active Session과 active mobile refresh credential이다.
- Authorization Source 조회는 로컬 transaction 밖에서 수행하고 commit 직전에 Session 상태를 다시 확인한다.
- credential row lock, 이전 credential 회전, 새 credential, AccessGrant snapshot, 감사 OutboxEvent를 한 트랜잭션에 저장한다.
- reuse가 탐지되면 Session과 refresh family를 폐기하고 보안 감사 event를 같은 트랜잭션에 저장한다.
- 새 access JWT는 저장 Entity가 아니며 commit된 결과에 대해서만 생성·전달한다.

## 멱등성과 동시성

- 멱등 범위는 API ID, refresh credential scope와 `Idempotency-Key`다.
- token 원문 대신 keyed digest와 canonical request HMAC fingerprint만 저장한다.
- 같은 token과 같은 key는 복구 TTL 동안 암호화한 최초 회전 결과를 재생한다.
- 같은 이전 token과 다른 key는 reuse로 판단해 Session과 family를 폐기한다.
- credential row lock으로 동시에 들어온 회전 요청 중 하나만 성공시킨다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ProblemDetails schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| Session 폐기 또는 token 재사용 | token 상태의 세부 정보를 숨긴다. | 로컬 token을 삭제하고 다시 로그인한다. |
| 사용자 제한 | 제한 세부 사유를 노출하지 않는다. | 지원 절차를 안내하고 재발급하지 않는다. |
| 같은-key replay TTL 만료 | 이전 token이나 응답을 다시 발급하지 않는다. | 다시 로그인한다. |
| 권한 원천 장애 | 낡은 권한으로 새 token을 만들지 않는다. | `Retry-After` 정책에 따라 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `RefreshSessionHandler` |
| Aggregate / Entity | `Session`, `SessionCredential`, `AccessGrant`, `UserAuthState` |
| Repository / Read Model | Session·Credential Repository, `IdempotencyRecord`와 replay payload store |
| Port / Adapter | Authorization Source, access JWT signer |
| Domain / Integration Event | Session refresh·reuse 탐지 감사 OutboxEvent |

## 관측성과 운영

- 로그에는 session ID, refresh family ID와 일반화된 결과만 남기고 token은 남기지 않는다.
- `auth_session_refresh_total`, `auth_refresh_reuse_detected_total`, refresh latency를 관측한다.
- Authorization Source, credential lock, transaction과 token signer를 별도 span으로 기록한다.
- refresh spike와 reuse 탐지는 보안 경보 대상으로 운영한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 정상 회전에서 이전 credential이 비활성화되고 새 credential 하나만 active가 된다.
- 같은 key 응답 유실 재시도에서 동일 token 묶음을 복구한다.
- 같은 이전 token의 다른-key 사용이 family 폐기로 이어진다.
- 동시 refresh 중 하나만 회전에 성공한다.
- token 원문과 개인정보가 저장소 메타데이터, 로그, trace, event에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.300-02 휴대폰 로그인과 refresh token 회전](../../../80-sequence/A_300_auth/SCN_A_300_02_phone_signin_refresh.md)
- 관련 API: `API.A.300-08`, `API.A.300-09`
- 여러 참여자의 Mermaid 다이어그램은 `80-sequence` 문서에서 관리한다.

## 호환성과 변경 정책

- token 만료 시각과 공개 가능한 Session metadata 추가는 하위 호환으로 처리한다.
- refresh credential의 전달 위치와 rotation 의미 변경은 새 API 버전으로 제공한다.
- JWT claim 변경은 issuer/audience consumer 계약과 별도로 versioning한다.

## 확인 필요

- 암호화 refresh replay TTL과 family 단위 폐기 보존 기간의 초기 운영값을 확정해야 한다.
