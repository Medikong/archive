---
id: API.A.300-03
title: 이메일 회원가입 시작 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, registration]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# API.A.300-03 이메일 회원가입 시작

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/registrations` |
| operationId | `startEmailRegistration` |
| 역할 | 이메일 회원가입 Registration과 이메일·휴대폰 Identity 예약을 만든다. |
| API 유형 | Command |
| 인증 | 웹 사전 인증 cookie와 CSRF·Origin 또는 모바일 auth flow token |
| 권한 | proof와 요청의 AuthenticationIntent binding이 일치해야 한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_03_start_email_registration.yaml](openapi/paths/API_A_300_03_start_email_registration.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

요청·응답 구조, required 필드, HTTP 상태, 오류와 wire 예시는 OpenAPI를 기준으로 한다.

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

- 보장하는 업무 결과: 가입 작업, 두 Identity 예약과 PasswordCredential을 생성하고 상태 조회 전용 proof를 발급한다.
- 요청 안에서 하지 않는 일: Challenge를 발급·발송하거나 `user_id`와 Session을 만들지 않는다.
- 다른 Context 또는 외부 시스템의 책임: BFF는 프로필·추천인과 필수 동의를 먼저 처리하고 opaque reference만 전달한다.

## 보안과 개인정보

- `profileRequestId`, `agreementReceiptId`, `rememberMe`를 포함한 필수 입력을 검증한다.
- 비밀번호는 즉시 memory-hard hash로 바꾸고 평문·복호화 가능 값·단순 hash를 저장하지 않는다.
- 이메일과 휴대폰은 정규화 조회 키와 암호화 표시값을 분리한다.
- `registrationStatusToken`은 API 28 조회만 허용하며 hash와 별도 만료 시각만 저장한다. 가입 완료나 Session 발급 권한으로 사용할 수 없다.
- 중복·예약·위험 판정은 하나의 공개 오류로 일반화한다.

## 처리 규칙

1. auth flow proof와 `authIntentId`의 channel·소유 binding을 확인한다.
2. Context reference 유효성, 이메일, 비밀번호 정책과 휴대폰 E.164 변환 가능 여부를 검증한다.
3. 기존 미귀속 예약을 잠금 재사용할 수 있는지 확인한다.
4. `childExpiresAt = min(now + Registration TTL, Intent 생성 시각 + auth-flow 최대 수명)`으로 계산한다.
5. Registration, 두 Identity와 PasswordCredential을 만들고 AuthenticationIntent와 active owner proof의 만료 시각을 `childExpiresAt`으로 맞춘다.
6. 상태 조회 전용 `registrationStatusToken`을 발급하고 hash·만료 시각만 Registration에 저장한다. Challenge는 후속 발급 API에서 만든다.

## 상태 변경과 트랜잭션

- 시작 상태: 가입 작업 없음 또는 cooldown을 지난 미귀속 Identity 예약.
- 성공 종료 상태: `pending_verification` Registration과 pending email·phone Identity.
- 실패·만료 상태: 검증 실패 시 아무 Aggregate도 만들지 않는다.
- Registration, Identity, PasswordCredential, status proof hash, IdempotencyRecord와 OutboxEvent를 저장하고 parent Intent·proof 만료를 같은 트랜잭션에서 연장한다.
- Registration과 parent Intent·proof는 같은 `childExpiresAt`을 사용하며 auth-flow 최대 수명을 넘지 않는다.
- 프로필·동의 Context와 이메일/SMS Provider를 트랜잭션 안에서 호출하지 않는다.

## 멱등성과 동시성

- 멱등 범위는 `API.A.300-03 + auth flow + Idempotency-Key`다.
- 비밀번호를 포함한 전체 canonical body의 keyed HMAC fingerprint만 저장한다.
- 같은 key·같은 요청은 같은 Registration을 유지하되 상태 조회 token과 hash를 원자적으로 교체하고 이전 token을 무효화한다.
- 같은 key 성공 응답을 다시 전달할 때도 상태 조회 token 원문이나 암호화된 응답 payload를 저장하지 않는다.
- 같은 key·다른 요청은 충돌하며, 이메일 lookup unique 제약으로 동시 예약을 직렬화한다.
- 완료 전 Identity 재사용은 row lock과 Registration 상태·cooldown을 확인한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 error code는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| 입력·reference·비밀번호 정책 오류 | 입력 원문과 내부 reference 상태를 노출하지 않는다. | 입력과 선행 Context 처리를 확인한다. |
| 식별자 사용 불가 | 중복, 예약과 위험 판정을 구분하지 않는다. | 다른 식별자를 사용하거나 지원 절차를 확인한다. |
| Intent 부재·만료 또는 서비스 장애 | 부분 Registration을 만들지 않는다. | 새 Intent 또는 공개된 재시도 기준으로 다시 시작한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `StartEmailRegistrationHandler` |
| Aggregate / Entity | `Registration`, `Identity`, `PasswordCredential`, `AuthenticationIntent` |
| Repository / Read Model | `RegistrationRepository`, `IdentityRepository`, `IdempotencyRecordRepository` |
| Port / Adapter | `IdentityProtectorPort`, `PasswordHasherPort`, `AuthenticationPolicyProvider`, `RateLimitPort` |
| Domain / Integration Event | 가입 시작 감사 OutboxEvent |

## 관측성과 운영

- 가입 시작 latency와 result, 식별자 사용 불가·rate-limit 비율을 일반화된 label로 집계한다.
- 이메일, 휴대폰, 비밀번호와 Context reference 원문을 로그·trace·metric에 남기지 않는다.
- 초기 제한은 IP와 normalized identifier 기준 시간당 5회다.
- 인증 요청·응답 body sampling은 비활성화한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 필수 reference와 `rememberMe`가 빠진 요청을 거부한다.
- 이메일 정규화와 휴대폰 E.164 변환을 검증한다.
- 같은 key replay는 같은 Registration과 새 상태 조회 token을 반환하며 다른 body는 충돌하고 이전 token은 거부한다.
- Registration TTL이 auth-flow 최대 수명을 넘지 않으며 parent Intent와 active proof가 Registration 만료 전 먼저 끝나지 않는다.
- 상태 조회 token은 API 28에만 사용할 수 있고 terminal 뒤 짧은 보존 기간이 끝나면 거부된다.
- 시작 성공만으로 Challenge 발송이나 IdentityLink·Session이 생성되지 않는다.
- 비밀번호와 연락처 원문이 멱등 저장소와 관측 데이터에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.300-01 이메일 회원가입과 자동 로그인](../../../80-sequence/A_300_auth/SCN_A_300_01_email_registration.md)
- 관련 API: `API.A.300-03~06`, `API.A.300-28`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- 선택 입력 추가는 기존 가입 조건과 개인정보 경계를 바꾸지 않는 경우에만 하위 호환으로 처리한다.
- 필수 reference, 이메일·휴대폰 의미와 비밀번호 정책 전달 방식 변경은 새 API 버전으로 처리한다.
- 비밀번호 정책 강화는 policy version과 클라이언트 안내 기간을 함께 운영한다.

## 확인 필요

- PasswordPolicy의 최소·최대 길이와 Registration TTL의 초기 운영값은 아직 확정되지 않았다.
