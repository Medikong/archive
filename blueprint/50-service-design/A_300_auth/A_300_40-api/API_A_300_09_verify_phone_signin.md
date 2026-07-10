---
id: API.A.300-09
title: 휴대폰 로그인 검증 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, phone-signin, session]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# API.A.300-09 휴대폰 로그인 검증

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/signins/phone/challenges/{challengeId}/verify` |
| operationId | `verifyPhoneSignIn` |
| 역할 | SMS code 확인 뒤 active IdentityLink의 `user_id`로 채널별 Session을 발급한다. |
| API 유형 | Command |
| 인증 | 웹 사전 인증 cookie와 CSRF·Origin 또는 모바일 auth flow token |
| 권한 | proof가 Challenge를 소유하고 연결된 UserAuthState가 active여야 한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_09_verify_phone_signin.yaml](openapi/paths/API_A_300_09_verify_phone_signin.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

code schema, 채널별 Session 응답, cookie·token과 오류 wire 예시는 OpenAPI를 기준으로 한다.

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

- 보장하는 업무 결과: code, IdentityLink, 사용자 상태와 권한을 확인하고 정확히 하나의 논리 Session을 발급한다.
- 요청 안에서 하지 않는 일: 휴대폰 번호로 새 사용자 계정을 만들거나 기존 계정을 병합하지 않는다.
- 다른 Context 또는 외부 시스템의 책임: Authorization Source가 최신 AccessGrant를 제공한다.

## 보안과 개인정보

- 6자리 SMS code를 keyed digest로 검증하고 원문을 저장·관측하지 않는다.
- code 검증 성공 뒤에만 active Link 부재와 잠금 상태를 공개한다.
- Session TTL은 API 08에서 AuthenticationIntent에 고정한 `rememberMe`와 Intent 생성 시 확정한 client channel을 사용한다.
- JWT, 내부 헤더와 Event에 이메일·휴대폰·Identity ID를 넣지 않는다.

## 처리 규칙

1. auth flow proof, Challenge와 AuthenticationIntent의 소유 binding을 확인하고 Intent의 client channel·`rememberMe`를 읽는다.
2. code를 검증하고 Challenge를 원자적으로 소비한다.
3. verified phone Identity의 잠금과 active IdentityLink·UserAuthState를 확인한다.
4. Authorization Source에서 최신 AccessGrant를 조회한다.
5. Session, credential, 실패 count 초기화, Intent 소비와 감사 OutboxEvent를 저장한다.

## 상태 변경과 트랜잭션

- 시작 상태: `issued` phone_signin Challenge.
- 성공 종료 상태: `verified` Challenge와 active Session·SessionCredential.
- 실패·만료 상태: code 실패·만료, Identity 잠금, Link 부재 또는 사용자 제한.
- Challenge 실패와 attempt·잠금 감사 상태를 원자적으로 저장한다.
- 성공은 Session 관련 상태와 outbox를 저장하며 Authorization Source 호출 중 트랜잭션을 열어 두지 않는다.

## 멱등성과 동시성

- 멱등 범위는 `API.A.300-09 + Challenge + Idempotency-Key`다.
- code를 포함한 canonical body의 keyed HMAC fingerprint만 저장한다.
- 같은 key 실패 replay는 Challenge와 Identity failure count를 다시 올리지 않는다.
- 성공 응답 유실은 consumed Intent의 recovery-only owner proof와 같은 Session·operation·key·request fingerprint를 일반 인증 전에 확인하고 verified Challenge 결과를 다시 검증한 뒤 같은 논리 Session의 credential만 교체한다. 응답 암호문은 저장하지 않는다.
- Challenge row lock과 Session unique 제약으로 중복 소비·발급을 막는다.

## 예외와 복구 규칙

정확한 HTTP 상태와 error code는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| code 실패·만료 | Link와 계정 존재 여부를 공개하지 않는다. | 올바른 code를 제출하거나 새 Challenge를 받는다. |
| code 확인 뒤 Link 부재·잠금·제한 | 검증된 번호 소유자에게만 일반화된 상태를 공개한다. | 회원가입·연동, 대기 또는 지원 절차를 따른다. |
| 권한 원천·저장소 장애 | Session을 부분 발급하지 않는다. | 같은 key로 재시도한다. |
| credential 전달 복구 TTL 종료 | 번호·Link·Session 세부 상태를 숨긴다. | `AUTH_SESSION_DELIVERY_EXPIRED` 뒤 새 로그인을 시작한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `VerifyChallengeHandler`, `SignInWithPhoneHandler`, `IssueSessionHandler` |
| Aggregate / Entity | `AuthenticationIntent`, `VerificationChallenge`, `Identity`, `IdentityLink`, `Session`, `SessionCredential`, `AccessGrant` |
| Repository / Read Model | Intent·Challenge·Identity·Link·Session·AccessGrant·Idempotency Repository |
| Port / Adapter | `CredentialIssuerPort`, Authorization Source, `AuthenticationPolicyProvider`, `RateLimitPort` |
| Domain / Integration Event | `EVT.A.300-05~07`, `EVT.A.300-31~33` |

## 관측성과 운영

- phone signin latency, code 실패, Link 부재, lock, Session 발급 결과를 일반화된 label로 집계한다.
- 가입·미가입 번호의 발급과 검증 시간 분포를 함께 감시한다.
- Challenge당 최대 검증 횟수와 대상·IP rate limit을 적용하고 `Retry-After`만 공개한다.
- code, 휴대폰, token과 cookie 원문의 body sampling을 비활성화한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- API 08이 AuthenticationIntent에 저장한 `rememberMe`와 Intent의 client channel로 발급 정책을 결정한다.
- `rememberMe=true` 모바일 Session의 만료가 refresh token 만료보다 빠르지 않다.
- 같은 key 실패 replay가 attempt와 lock count를 다시 올리지 않는다.
- 올바른 code 뒤 active Link가 없으면 Session을 만들지 않는다.
- 잠금 상태는 code 확인 뒤에만 공개한다.
- 응답 유실 복구 후에도 논리 Session은 하나다.
- consumed owner proof는 정확한 복구 binding에서만 성공하고 다른 API에서는 거부된다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.300-02 휴대폰 로그인과 refresh token 회전](../../../80-sequence/A_300_auth/SCN_A_300_02_phone_signin_refresh.md)
- 관련 API: `API.A.300-08`, `API.A.300-09`, `API.A.300-14`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- 성공 응답의 선택 metadata는 credential 분기 의미를 유지할 때만 하위 호환으로 추가한다.
- code 형식과 Challenge binding 변경은 새 API 버전으로 처리한다.
- Link 부재를 code 검증 전에 공개하는 변경은 허용하지 않는다.

## 확인 필요

- phone signin 실패를 Identity 전역 잠금에 반영하는 세부 window와 lock duration은 확정되지 않았다.
