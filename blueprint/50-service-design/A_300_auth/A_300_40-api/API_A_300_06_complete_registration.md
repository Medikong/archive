---
id: API.A.300-06
title: 회원가입 완료 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, registration, session]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# API.A.300-06 회원가입 완료

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/registrations/{registrationId}/complete` |
| operationId | `completeRegistration` |
| 역할 | 가입 인증 완료를 수락하고 Context 사용자 계정 연동 뒤 자동 로그인을 완료한다. |
| API 유형 | 장기 실행 Command |
| 인증 | 웹 사전 인증 cookie와 CSRF·Origin 또는 모바일 auth flow token |
| 권한 | proof가 Registration을 소유하고 두 필수 Challenge가 verified여야 한다. |
| 노출 범위 | public |
| 멱등성 | 최초 `Idempotency-Key`를 수락부터 Session 전달까지 계속 사용 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_06_complete_registration.yaml](openapi/paths/API_A_300_06_complete_registration.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

최초 수락, 상태 재개, 채널별 credential 응답과 모든 오류 wire 계약은 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [SCN.A.300-01](../../../80-sequence/A_300_auth/SCN_A_300_01_email_registration.md) |

## 책임과 경계

- 보장하는 업무 결과: 가입 인증 snapshot 통지, Context 사용자의 계정 연동 반영과 클라이언트 요청에서의 Session 발급을 단계별로 완료한다.
- 요청 안에서 하지 않는 일: Auth가 `user_id`를 생성하거나 Context 사용자에 동기 발급 요청하지 않는다.
- 다른 Context 또는 외부 시스템의 책임: Context 사용자가 사용자 계정과 `user_id`를 만들고 계정 연동 Event를 발행한다.

## 보안과 개인정보

- Registration 소유 proof, 웹 CSRF·Origin과 두 Challenge의 verified binding을 확인한다.
- credential 전달 채널은 요청 헤더가 아니라 Registration에 저장된 client channel을 사용한다.
- 웹 성공 응답은 Session cookie 설정과 auth-flow cookie 폐기를 같은 `Set-Cookie` 응답에 포함한다.
- `rememberMe=true`인 모바일 Session의 논리 만료는 refresh token 만료보다 빠르지 않다.
- Event와 로그에 이메일, 휴대폰, code, 비밀번호와 Session credential 원문을 넣지 않는다.
- 제한 사용자 상태는 Identity 귀속을 유지하되 Session을 발급하지 않는다.

## 처리 규칙

1. 최초 요청에서 두 필수 Challenge와 Identity snapshot을 다시 검증한다.
2. verification binding, processing IdempotencyRecord와 가입 인증 완료 OutboxEvent를 저장하고 계정 연동 대기를 수락한다.
3. Context 사용자의 binding된 연동 요청이 적용될 때까지 상태 조회로 진행 상태를 확인한다.
4. `linked` 뒤 같은 key 요청에서 Authorization Source의 최신 AccessGrant를 조회한다.
5. 클라이언트 요청에서만 Session credential을 만들고 채널별로 전달한다.

## 상태 변경과 트랜잭션

- 시작 상태: `pending_verification` 또는 재개 가능한 `awaiting_user_link`, `linked`, `issuing_session`.
- 성공 종료 상태: `completed` Registration과 정확히 하나의 논리 Session.
- 실패·만료 상태: 사용자 연동 거부, link 수락 기한 또는 Session 전달 기한 종료.
- 최초 트랜잭션은 binding, Registration, IdempotencyRecord와 가입 완료 OutboxEvent를 저장한다.
- 최종 트랜잭션은 Session, credential, AccessGrant, Intent 소비, Registration과 OutboxEvent를 저장한다.
- Event consumer와 상태 조회는 credential을 발급하지 않는다.

## 멱등성과 동시성

- 최초 완료 key를 Registration에 고정하고 최종 응답까지 같은 key만 허용한다.
- 같은 key replay는 현재 Registration 상태를 반환하거나 다음 안전한 단계를 재개한다.
- 다른 key는 두 번째 Event·Link·Session을 만들지 않고 충돌한다.
- 응답 유실 복구는 consumed Intent의 recovery-only owner proof와 같은 Session·operation·key·request fingerprint를 일반 인증 전에 검증하고, 같은 논리 Session의 SessionCredential만 교체한다. credential 응답 암호문은 저장하지 않는다.
- Registration version, verification binding, Inbox event ID와 link request ID를 각각 검증한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 error code는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| 필수 소유 확인 미완료 | 누락된 인증 수단만 일반적으로 안내한다. | 이메일·휴대폰 검증을 완료한다. |
| 사용자 연동 대기·거부·기한 종료 | 내부 Event와 사용자 생성 사유를 숨긴다. | 상태 조회, 새 가입 또는 일반 로그인을 선택한다. |
| Authorization Source 장애 | 부분 Session을 발급하지 않는다. | 같은 key로 Session 발급 단계부터 재개한다. |
| credential 전달 복구 TTL 종료 | 이미 연결된 IdentityLink는 유지한다. | `AUTH_SESSION_DELIVERY_EXPIRED` 뒤 일반 로그인으로 진행한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `AdvanceRegistrationHandler`, `ApplyRegistrationUserLinkHandler` |
| Aggregate / Entity | `Registration`, `RegistrationVerificationBinding`, `IdentityLink`, `Session`, `SessionCredential` |
| Repository / Read Model | `RegistrationRepository`, `IdentityLinkRepository`, `SessionRepository`, `IdempotencyRecordRepository`, Inbox/Outbox Repository |
| Port / Adapter | `AuthorizationSource`, `CredentialIssuerPort`, Event Broker relay |
| Domain / Integration Event | `Auth.RegistrationVerificationCompleted`, `User.AuthLinkRequested`, `EVT.A.300-35~37`, `EVT.A.300-24` |

## 관측성과 운영

- 상태별 체류 시간, link backlog age, Session 발급 latency와 재개 횟수를 관측한다.
- 로그·trace에는 Registration과 Event reference, 상태, 일반화된 결과만 남긴다.
- Context 사용자 지연은 `awaiting_user_link`를 유지하고 기한 초과를 경보한다.
- Authorization Source timeout 초기값은 500ms이며 새 Session 발급은 fail closed다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 두 Challenge가 verified가 아니면 가입 완료 Event를 만들지 않는다.
- 중복·역순 Event가 IdentityLink와 Session을 중복 생성하지 않는다.
- 최초 `202` 뒤 다른 key 요청을 거부한다.
- 상태 조회와 Event consumer가 credential을 발급하지 않는다.
- 최종 응답 유실 복구 후에도 논리 Session이 하나다.
- recovery-only owner proof는 정확한 복구 binding에서만 성공하고 다른 API에서는 거부된다.
- 웹 성공 시 Session cookie가 설정되고 auth-flow cookie가 폐기되며, 모바일 예시의 Session·refresh token 만료가 일치한다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.300-01 이메일 회원가입과 자동 로그인](../../../80-sequence/A_300_auth/SCN_A_300_01_email_registration.md)
- 관련 API: `API.A.300-03~06`, `API.A.300-28`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- 진행 상태 추가는 기존 상태 의미와 terminal 결과를 바꾸지 않을 때 하위 호환으로 처리한다.
- Event version, binding 필드와 credential 전달 구조 변경은 producer·consumer 동시 이행 계획과 새 버전이 필요하다.
- 장기 실행 재개 정책 변경은 기존 processing record의 TTL을 소급해 단축하지 않는다.

## 확인 필요

- link accept TTL, Inbox processing grace와 자동 Session 전달 TTL의 초기 운영값은 확정되지 않았다.
