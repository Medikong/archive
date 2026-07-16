---
id: API.A.300-08
title: 휴대폰 로그인 Challenge 발급 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, phone-signin, challenge]
source: local
created: 2026-07-10
updated: 2026-07-16
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# API.A.300-08 휴대폰 로그인 Challenge 발급

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/signins/phone/challenges` |
| operationId | `issuePhoneSignInChallenge` |
| 역할 | 휴대폰 로그인용 SMS Challenge를 만들고 접수 결과를 동일한 형태로 반환한다. |
| API 유형 | Command |
| 인증 | 웹 사전 인증 cookie와 CSRF·Origin 또는 모바일 auth flow token |
| 권한 | proof와 요청의 AuthenticationIntent binding이 일치해야 한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_08_issue_phone_signin_challenge.yaml](openapi/paths/API_A_300_08_issue_phone_signin_challenge.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

전화번호 schema, 접수 응답, 오류와 wire 예시는 OpenAPI를 기준으로 한다.

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

- 보장하는 업무 결과: abuse control을 통과한 모든 정규화 번호에 실제 Challenge와 SMS delivery를 만들고 같은 접수 응답을 반환한다.
- 요청 안에서 하지 않는 일: IdentityLink 존재 여부를 공개하거나 Session을 발급하지 않는다.
- 다른 Context 또는 외부 시스템의 책임: relay와 선택된 SMS Adapter가 계정 연결 여부와 무관하게 메시지를 전달한다.

## 보안과 개인정보

- 휴대폰 번호는 공통 `PhoneNumber` 입력을 E.164로 정규화한다.
- 발급 단계에서는 IdentityLink를 조회하지 않으며 가입 여부에 따라 응답 형태·상태·SMS 발송 여부를 바꾸지 않는다.
- 휴대폰 원문과 lookup key, code와 `rememberMe`를 로그·trace·metric·Event에 넣지 않는다.
- 첫 발급 요청의 `rememberMe`를 AuthenticationIntent에 고정하고 이후 같은 Intent 요청은 저장값과 일치해야 한다.

## 처리 규칙

1. auth flow proof와 `authIntentId` binding을 확인한다.
2. PhoneNumber를 정규화하고 rate limit을 확인한다.
3. 첫 요청이면 `rememberMe`를 AuthenticationIntent에 기록하고, 이미 기록되어 있으면 요청값과 client channel이 일치하는지 확인한다.
4. IdentityLink 조회 없이 번호에 묶인 실제 `phone_signin` Challenge와 delivery outbox를 저장한다.
5. 계정 연결 여부와 무관하게 동일한 accepted 응답을 반환한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `AuthenticationIntent`, `VerificationChallenge`, `IdempotencyRecord`, `OutboxEvent` | 우회 | Challenge 재발급과 `rememberMe` 고정은 PostgreSQL에서 원자 처리한다. |
| `AuthenticationPolicySnapshotProjection` (`P`) | 사용 | SMS 발급, 재발송 제한과 Challenge TTL은 반복 조회되는 정책이다. |
| Challenge·전화번호 | 사용하지 않음 | 계정 연결 여부와 code 관련 값을 Redis에 복제하면 계정 열거와 비밀 노출 위험만 커진다. |

## 상태 변경과 트랜잭션

- 시작 상태: active AuthenticationIntent와 Challenge 없음 또는 재발급 가능한 이전 Challenge.
- 성공 종료 상태: `issued` Challenge, delivery outbox와 AuthenticationIntent에 저장된 `rememberMe`.
- 실패·만료 상태: Intent 만료나 rate limit이면 Challenge를 만들지 않는다.
- AuthenticationIntent의 `rememberMe`, Challenge, IdempotencyRecord와 delivery outbox를 같은 트랜잭션에 저장한다.
- Provider 호출은 relay가 커밋 뒤 수행한다.

## 멱등성과 동시성

- 멱등 범위는 `API.A.300-08 + auth flow + Idempotency-Key`다.
- 전화번호를 포함한 canonical body의 keyed HMAC fingerprint만 저장한다.
- 같은 key는 같은 Challenge와 accepted 응답을 재생하고 발송을 반복하지 않는다.
- 같은 key·다른 번호, Intent 또는 `rememberMe`는 충돌하고, 다른 key라도 같은 Intent의 `rememberMe` 변경 요청은 거부한다.
- context와 purpose별 active Challenge unique 제약으로 동시 중복 발급을 막는다.

## 예외와 복구 규칙

정확한 HTTP 상태와 error code는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| 가입·미가입 번호 | 모두 실제 SMS를 발송하고 같은 접수 응답을 사용한다. | 받은 Challenge 화면에서 code를 입력한다. |
| Intent 만료 | 번호 존재 여부를 추가로 공개하지 않는다. | 새 Intent를 만든다. |
| 발급 제한 또는 장애 | 대상과 threshold를 숨긴다. | `Retry-After` 이후 새 key로 요청한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `RequestVerificationChallengeHandler` |
| Aggregate / Entity | `AuthenticationIntent`, `VerificationChallenge` |
| Repository / Read Model | `AuthenticationIntentRepository`, `VerificationChallengeRepository`, `IdempotencyRecordRepository` |
| Port / Adapter | `IdentityProtectorPort`, `RateLimitPort`, `SmsVerificationSender` |
| Domain / Integration Event | `EVT.A.300-22`, `VerificationDeliveryRequested` |

## 관측성과 운영

- Challenge 발급 수와 delivery relay 성공률을 감시하되 가입 여부 label은 만들지 않는다.
- 번호별 응답 시간 분포의 이상치와 abuse-control 거부율을 보안 지표로 감시한다.
- 초기 제한은 phone lookup key와 IP 기준 15분 3회다.
- 휴대폰·code 원문과 body sampling을 관측 시스템에서 제외한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- `rememberMe`와 공통 PhoneNumber가 없는 요청을 거부한다.
- 가입·미가입 번호의 응답 schema와 상태가 같다.
- 같은 key replay가 Challenge와 SMS 발송을 중복하지 않는다.
- active Link가 없어도 실제 Challenge와 delivery outbox를 만들며 발급 단계에서 Link를 조회하지 않는다.
- 첫 요청의 `rememberMe`가 AuthenticationIntent에 저장되고 이후 불일치 요청은 거부된다.
- 휴대폰 원문과 계정 존재 여부가 관측 데이터에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.300-02 휴대폰 로그인과 refresh token 회전](../A_300_50-sequence/SCN_A_300_02_phone_signin_refresh.md)
- 관련 API: `API.A.300-08`, `API.A.300-09`, `API.A.300-14`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- PhoneNumber에 선택 metadata를 추가할 때 E.164 의미와 기존 필수값을 유지한다.
- `rememberMe`의 필수 여부와 AuthenticationIntent binding 변경은 새 API 버전으로 처리한다.
- 실제 발송 여부를 드러내는 응답 추가는 허용하지 않는다.

## 확인 필요

- Challenge TTL과 abuse-control threshold의 초기 운영값은 확정되지 않았다.
