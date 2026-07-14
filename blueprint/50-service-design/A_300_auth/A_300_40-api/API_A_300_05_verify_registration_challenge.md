---
id: API.A.300-05
title: 가입 Challenge 검증 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, registration, challenge]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# API.A.300-05 가입 Challenge 검증

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/registrations/{registrationId}/challenges/{challengeId}/verify` |
| operationId | `verifyRegistrationChallenge` |
| 역할 | 가입용 이메일·휴대폰 code를 검증하고 Identity 소유 확인 상태를 갱신한다. |
| API 유형 | Command |
| 인증 | 웹 사전 인증 cookie와 CSRF·Origin 또는 모바일 auth flow token |
| 권한 | proof가 Registration과 Challenge를 모두 소유해야 한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_05_verify_registration_challenge.yaml](openapi/paths/API_A_300_05_verify_registration_challenge.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

code schema, 성공 상태, 오류와 wire 예시는 OpenAPI를 기준으로 한다.

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

- 보장하는 업무 결과: Challenge를 한 번 소비하고 해당 email 또는 phone Identity를 verified로 바꾼다. 두 필수 Challenge가 모두 완료되면 짧은 수명의 `registrationCompletionProof`를 반환한다.
- 요청 안에서 하지 않는 일: `user_id`, IdentityLink와 Session을 만들지 않는다.
- 다른 Context 또는 외부 시스템의 책임: 가상 Adapter는 개발 환경에서 code 전달만 담당한다.

## 보안과 개인정보

- 이메일과 휴대폰 모두 MVP에서는 6자리 code 입력만 지원한다.
- code 원문은 저장·로그·trace·metric·Event에 남기지 않고 keyed digest로 검증한다.
- 소유 binding 불일치와 리소스 부재는 Registration 존재 여부를 더 공개하지 않는다.
- 남은 시도 횟수와 내부 잠금 threshold는 기본 공개 응답에 넣지 않는다.

## 처리 규칙

1. Registration·Challenge·auth flow binding과 purpose를 확인한다.
2. Challenge가 issued이고 만료 전인지 확인한다.
3. code를 constant-time으로 검증한다.
4. 성공하면 Challenge를 verified로 소비하고 대상 Identity와 Registration의 검증 상태를 갱신한다.
5. 두 필수 Identity가 모두 verified이면 registration ID, 검증 완료 시각과 만료를 묶은 서명 proof를 반환한다.
6. 실패하면 허용 범위에서 attempt count를 저장한다.

## 상태 변경과 트랜잭션

- 시작 상태: `issued` Challenge와 `pending_verification` Registration.
- 성공 종료 상태: `verified` Challenge와 verified Identity. 두 필수 수단이 완료되면 Registration은 `verified`다.
- 실패·만료 상태: 실패 횟수 초과 시 `failed`, TTL 경과 시 `expired`.
- Challenge, Identity, Registration과 IdempotencyRecord를 같은 트랜잭션에 저장한다.
- 외부 Provider 호출은 없다.

## 멱등성과 동시성

- 멱등 범위는 `API.A.300-05 + Registration + Challenge + Idempotency-Key`다.
- code를 포함한 canonical body의 keyed HMAC fingerprint만 저장한다.
- 같은 key의 성공과 실패는 이전 결과를 재생하고 attempt count를 다시 올리지 않는다.
- 같은 key·다른 code는 충돌한다.
- Challenge version과 row lock으로 동시 검증 중 하나만 terminal 상태를 만든다.

## 예외와 복구 규칙

정확한 HTTP 상태와 error code는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| code 불일치 | 계정·Identity와 남은 횟수를 숨긴다. | 같은 Challenge에 새 key로 올바른 code를 제출한다. |
| Challenge 만료 | secret과 내부 만료 사유를 숨긴다. | 발급 API에서 새 Challenge를 요청한다. |
| 멱등 충돌 또는 서비스 장애 | terminal 상태를 덮어쓰지 않는다. | 원래 key를 사용하거나 공개된 기준으로 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `VerifyChallengeHandler`, `VerifyEmailChallengeHandler`, `VerifyPhoneChallengeHandler` |
| Aggregate / Entity | `Registration`, `VerificationChallenge`, `Identity` |
| Repository / Read Model | `RegistrationRepository`, `VerificationChallengeRepository`, `IdentityRepository`, `IdempotencyRecordRepository` |
| Port / Adapter | `AuthenticationPolicyProvider`, `Clock` |
| Domain / Integration Event | `EVT.A.300-04`, `EVT.A.300-05`, `EVT.A.300-23`, `EVT.A.300-33` |

## 관측성과 운영

- purpose, channel, result와 일반화된 reason category를 기록한다.
- 검증 latency, 실패·만료·rate-limit 비율과 attempt limit 도달 수를 집계한다.
- code 원문과 hash를 로그·trace·metric label에 넣지 않는다.
- 인증 Endpoint의 body sampling은 비활성화한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 6자리 이외 code와 추가 속성을 거부한다.
- 같은 key 실패 replay가 attempt count를 다시 증가시키지 않는다.
- 동시 성공 요청 중 하나만 Challenge를 소비한다.
- 이메일과 휴대폰 모두 verified이면 프론트엔드가 User를 생성할 수 있는 `registrationCompletionProof`를 반환한다.
- code 원문과 digest가 관측 데이터에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.300-01 이메일 회원가입과 자동 로그인](../../../80-sequence/A_300_auth/SCN_A_300_01_email_registration.md)
- 관련 API: `API.A.300-03~06`, `API.A.300-28`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- code 길이와 문자 집합 변경은 기존 클라이언트 입력 계약에 영향을 주므로 새 버전으로 처리한다.
- link token proof 도입은 request `oneOf`와 별도 검증 정책을 추가하는 계약 변경이다.
- 새 verification method는 기존 method 의미를 바꾸지 않고 추가한다.

## 확인 필요

- VerificationPolicy의 최대 시도 횟수와 code TTL 최종값은 확정되지 않았다.
