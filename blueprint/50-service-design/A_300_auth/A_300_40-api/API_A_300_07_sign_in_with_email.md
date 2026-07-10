---
id: API.A.300-07
title: 이메일 로그인 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, signin, session]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# API.A.300-07 이메일 로그인

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/signins/email` |
| operationId | `signInWithEmail` |
| 역할 | 이메일·비밀번호를 검증하고 채널별 Session credential을 발급한다. |
| API 유형 | Command |
| 인증 | 웹 사전 인증 cookie와 CSRF·Origin 또는 모바일 auth flow token |
| 권한 | active IdentityLink와 UserAuthState, 최신 AccessGrant를 확인한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_07_sign_in_with_email.yaml](openapi/paths/API_A_300_07_sign_in_with_email.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

요청 credential, 채널별 성공 응답, cookie·token과 오류 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | 직접 연결된 이메일 로그인 시퀀스는 아직 없다. |

## 책임과 경계

- 보장하는 업무 결과: credential, Link, 사용자 상태와 권한을 확인하고 정확히 하나의 논리 Session을 발급한다.
- 요청 안에서 하지 않는 일: 사용자 프로필을 조회하거나 계정·Identity를 병합하지 않는다.
- 다른 Context 또는 외부 시스템의 책임: Authorization Source가 최신 role·permission snapshot을 제공한다.

## 보안과 개인정보

- 이메일 미존재와 비밀번호 불일치를 하나의 로그인 실패로 합치고 dummy password verification을 수행한다.
- 정확한 credential 확인 뒤에만 잠금, 강제 재설정과 사용자 제한 상태를 구분한다.
- 이메일, 비밀번호, token과 cookie 원문을 로그·trace·metric·Event에 넣지 않는다.
- 웹은 Session cookie와 CSRF token을 받고 auth-flow cookie를 즉시 폐기하며, 모바일은 access JWT와 opaque refresh token만 받는다.
- `rememberMe=true`인 모바일 Session의 논리 만료는 refresh token 만료보다 빠르지 않다.

## 처리 규칙

1. auth flow proof와 요청 Intent binding을 확인한다.
2. 정규화 이메일로 Identity와 active PasswordCredential을 찾고 비밀번호를 검증한다.
3. 잠금, 강제 재설정, active IdentityLink와 UserAuthState를 확인한다.
4. Authorization Source에서 최신 AccessGrant를 조회한다.
5. 실패 count 초기화, Session·credential, Intent 소비와 감사 OutboxEvent를 저장한다.

## 상태 변경과 트랜잭션

- 시작 상태: active 사전 인증 Intent와 로그인 가능한 email Identity.
- 성공 종료 상태: active Session·SessionCredential, 소비된 Intent와 초기 AccessGrant.
- 실패·만료 상태: 실패 횟수와 잠금 상태만 정책에 따라 갱신하며 Session은 만들지 않는다.
- 로그인 실패는 failure count와 감사 outbox를, 성공은 Session 관련 상태와 outbox를 각각 원자적으로 저장한다.
- Authorization Source 호출 중 DB 트랜잭션을 열어 두지 않고 이후 version을 다시 확인한다.

## 멱등성과 동시성

- 멱등 범위는 `API.A.300-07 + auth flow + Idempotency-Key`다.
- 비밀번호를 포함한 전체 canonical body의 keyed HMAC fingerprint만 저장한다.
- 같은 key의 완료된 실패는 비밀번호 검증과 failure count 증가 없이 재생한다.
- 성공 응답 유실 재시도는 consumed Intent의 recovery-only owner proof와 같은 Session·operation·key·request fingerprint를 일반 인증 전에 확인하고 비밀번호를 다시 검증한 뒤 논리 Session의 credential만 교체한다. 응답 암호문은 저장하지 않는다.
- Identity와 Session version, active credential unique 제약으로 중복 발급을 방지한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 error code는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| 이메일·비밀번호 검증 실패 | 계정 존재 여부를 숨긴다. | 입력을 확인하거나 재설정을 시작한다. |
| Intent 부재·만료 | credential 검증 전 상태를 더 공개하지 않는다. | 새 AuthenticationIntent를 만든다. |
| 잠금·강제 재설정·사용자 제한 | credential 확인 전에는 구분하지 않는다. | 공개 안내에 따라 대기, 재설정 또는 지원 절차를 진행한다. |
| 권한 원천·저장소 장애 | Session을 부분 발급하지 않는다. | `Retry-After`에 따라 같은 key로 재시도한다. |
| credential 전달 복구 TTL 종료 | 계정과 Session 세부 상태를 숨긴다. | `AUTH_SESSION_DELIVERY_EXPIRED` 뒤 새 로그인을 시작한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `SignInWithEmailHandler`, `IssueSessionHandler` |
| Aggregate / Entity | `Identity`, `PasswordCredential`, `IdentityLink`, `Session`, `SessionCredential`, `AccessGrant`, `AuthenticationIntent` |
| Repository / Read Model | `IdentityRepository`, `IdentityLinkRepository`, `SessionRepository`, `AccessGrantRepository`, `IdempotencyRecordRepository` |
| Port / Adapter | `PasswordHasherPort`, `CredentialIssuerPort`, Authorization Source, `RateLimitPort` |
| Domain / Integration Event | `EVT.A.300-06`, `EVT.A.300-07`, `EVT.A.300-31`, `EVT.A.300-32` |

## 관측성과 운영

- method, result, 일반화된 reason, policy version과 request ID만 관측한다.
- 로그인 p50/p95/p99, 실패율, lock rate, Authorization Source latency를 집계한다.
- 로그인 실패 제한은 Identity와 IP 보조 기준이며 초기 5회 뒤 잠금 정책을 적용한다.
- body sampling은 비활성화하고 예상 가능한 잘못된 비밀번호에 ERROR stack을 남기지 않는다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 미존재 이메일과 잘못된 비밀번호의 응답 schema와 시간 분포가 유사하다.
- 같은 key 실패 replay가 failure count를 다시 올리지 않는다.
- 잠금·강제 재설정·제한 상태에서 Session을 만들지 않는다.
- 성공 응답 유실 복구에도 논리 Session은 하나다.
- consumed owner proof는 정확한 복구 binding에서만 성공하고 일반 사전 인증 API에서는 거부된다.
- 웹 응답에 token이, 모바일 응답에 cookie·CSRF가 섞이지 않는다.
- 웹 성공 시 auth-flow cookie가 폐기되고, `rememberMe=true` 모바일 Session 만료가 refresh token 만료와 일치한다.

## 연관 시퀀스

- 직접 연결된 이메일 로그인 시퀀스 문서는 아직 없다.
- Intent 생성과 인증 후 행동 복구 API가 이 로그인 전후에 참여한다.
- 여러 참여자의 Mermaid 다이어그램은 [인증 시퀀스 인덱스](../../../80-sequence/A_300_auth/README.md)에서 관리한다.

## 호환성과 변경 정책

- 성공 응답의 새 선택 metadata는 기존 credential 분기를 바꾸지 않을 때 하위 호환으로 추가한다.
- 로그인 실패 구분을 더 상세하게 공개하는 변경은 보안 검토와 새 버전이 필요하다.
- 비밀번호·Session 정책 변경은 policy version으로 운영하고 이미 발급된 artifact를 소급 연장하지 않는다.

## 확인 필요

- LoginLockPolicy의 failure window와 lock duration 초기 운영값은 확정되지 않았다.
