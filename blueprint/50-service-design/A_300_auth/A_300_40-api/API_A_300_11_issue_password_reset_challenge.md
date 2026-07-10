---
id: API.A.300-11
title: 비밀번호 재설정 Challenge 발급 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-10
---

# API.A.300-11 비밀번호 재설정 Challenge 발급

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/password-resets/{passwordResetId}/challenges` |
| operationId | `issuePasswordResetChallenge` |
| 역할 | 선택한 `email` 또는 `phone` 인증 수단의 비밀번호 재설정 Challenge를 발급한다. |
| API 유형 | Command |
| 인증 | 웹 사전 인증 cookie 또는 모바일 auth flow token으로 PasswordReset 소유 증명 |
| 권한 | PasswordReset에 바인딩된 사전 인증 컨텍스트와 요청 채널 일치 |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_11_issue_password_reset_challenge.yaml](openapi/paths/API_A_300_11_issue_password_reset_challenge.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

Method/Path, parameter, request/response schema, content type, 응답 header, HTTP 상태, 오류 body와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [SCN.A.310-01](../../../80-sequence/A_300_auth/SCN_A_310_01_password_reset.md) |

## 책임과 경계

- PasswordReset의 소유 증명과 지원 채널을 확인하고 Challenge reference를 발급한다.
- 실제 재설정 대상과 선택 수단이 연결된 경우에만 delivery outbox를 저장한다.
- 외부 `method=phone`은 서버 내부에서 SMS 전달 방식으로 변환하며, 이메일·SMS Provider를 요청 thread에서 직접 호출하지 않는다.
- 계정 존재 여부와 연결 수단 유무를 공개 응답으로 구분하지 않는다.

## 보안과 개인정보

- 웹은 auth-flow cookie, CSRF token, 허용 Origin을 함께 검증하고 모바일은 auth flow token을 검증한다.
- 이메일, 휴대폰 번호, 인증번호와 연결 여부를 응답·로그·trace·metric label에 남기지 않는다.
- decoy PasswordReset도 실재 대상과 구분할 수 없는 Challenge reference와 응답 시간을 사용한다.
- 정확한 security AND/OR 조합은 OpenAPI를 따른다.

## 처리 규칙

1. PasswordReset 소유 증명, 상태, 만료와 요청 채널을 검증한다.
2. 선택한 `email` 또는 `phone` method가 PasswordResetPolicy에서 허용되는지 확인한다.
3. 실재 대상이면 `password_reset` 목적의 Challenge를 만들고 `phone`은 SMS delivery outbox로 변환한다.
4. decoy 대상이면 외부에서 구분할 수 없는 검증 불가능 Challenge reference를 만든다.
5. 발급 접수 결과만 반환하고 실제 전달은 Outbox Relay가 수행한다.

## 상태 변경과 트랜잭션

- 시작 상태는 `PasswordReset.requested`다.
- 실재 대상의 Challenge, PasswordReset 참조, delivery payload와 OutboxEvent를 한 트랜잭션에 저장한다.
- decoy 대상은 외부 계약을 유지하는 Challenge reference만 저장하고 delivery outbox를 만들지 않는다.
- 같은 context의 기존 issued Challenge를 교체하면 기존 항목을 `revoked`로 닫는다.
- Provider 호출은 commit 이후 relay가 수행하며 발송 실패가 PasswordReset 소유 확인 성공을 뜻하지 않는다.

## 멱등성과 동시성

- 멱등 범위는 API ID, PasswordReset 소유 컨텍스트와 `Idempotency-Key`다.
- request body는 서버 비밀키 HMAC fingerprint만 저장하고 원문 식별 정보는 저장하지 않는다.
- 같은 key와 같은 요청은 기존 Challenge metadata를 반환하고 메시지를 다시 발송하지 않는다.
- 같은 key와 다른 요청은 멱등 충돌로 거부한다.
- context와 purpose별 active Challenge 유일성 및 Challenge row version으로 동시 발급을 제어한다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ProblemDetails schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| PasswordReset 소유·상태 검증 실패 | 대상 존재 여부를 구분하지 않는다. | 비밀번호 재설정을 처음부터 다시 시작한다. |
| 발급 제한 초과 | 제한 기준과 내부 식별자를 숨긴다. | `Retry-After` 뒤 새 key로 재시도한다. |
| 저장소 또는 필수 의존성 장애 | 성공 형태로 응답하지 않는다. | 응답 정책에 따라 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `RequestVerificationChallengeHandler` |
| Aggregate / Entity | `PasswordReset`, `VerificationChallenge` |
| Repository / Read Model | PasswordReset·Challenge Repository, `IdempotencyRecord` |
| Port / Adapter | `EmailVerificationSender`, `SmsVerificationSender`, Outbox Relay |
| Domain / Integration Event | `VerificationDeliveryRequested`, 보안 감사 OutboxEvent |

## 관측성과 운영

- 로그에는 API ID, request ID, challenge ID, purpose, channel과 일반화된 결과만 남긴다.
- `auth_challenge_issued_total`, `auth_delivery_total`, rate-limit 결과를 관측한다.
- API, transaction, outbox relay와 Adapter 호출을 별도 span으로 기록한다.
- 실제 destination과 delivery payload는 body sampling 대상에서 제외한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 외부 method는 `email|phone`만 허용하고 `phone`을 내부 SMS 전달 방식으로 변환한다.
- 실재 대상과 decoy 대상의 상태, 응답 형태와 시간 분포가 구분되지 않는다.
- 같은 key 재요청이 Challenge나 발송을 중복 생성하지 않는다.
- Challenge와 outbox 저장 중 하나가 실패하면 둘 다 commit되지 않는다.
- 계정 식별자와 인증번호가 응답, 로그, trace, metric label에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.310-01 비밀번호 재설정](../../../80-sequence/A_300_auth/SCN_A_310_01_password_reset.md)
- 관련 API: `API.A.300-10`, `API.A.300-12`, `API.A.300-13`
- 여러 참여자의 Mermaid 다이어그램은 `80-sequence` 문서에서 관리한다.

## 호환성과 변경 정책

- 인증 수단 method 추가는 기존 enum 의미와 계정 존재 여부 보호를 유지할 때 하위 호환으로 추가한다.
- Challenge proof 방식과 Path 변경은 새 API 버전으로 제공한다.
- 기존 채널 제거는 정책 비활성화와 deprecation 기간을 거친다.

## 확인 필요

- decoy Challenge reference의 보존 기간과 정리 기준은 운영 정책에서 확정한다.
