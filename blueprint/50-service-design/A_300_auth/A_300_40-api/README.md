---
id: SD.A.30040
title: Context 인증 API 설계
type: service-design-api
status: draft
tags: [service-design, auth, api, session, token, password-reset, identity-link]
source: local
created: 2026-07-09
updated: 2026-07-10
service_design: SD.A.300
bounded_context: BC.A.300
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# Context 인증 API 설계

## 기본 정보

- Service Design ID: `SD.A.30040`
- API Base URL: `/api/v1/auth`
- 운영자 API Base URL: `/api/v1/operator/auth`
- 개발·테스트 API Base URL: `/api/v1/dev/auth`
- Canonical UC: [UC.A.300](../../../30-uc/UC_A_300_auth_member.md)
- 역할: Context 인증의 REST API, 이벤트 계약, 요청/응답, 오류, 보안 경계를 정의한다.
- 주요 클라이언트: 웹 프론트엔드, iOS/Android 앱, 운영자 사이트.
- 제외 범위: 사용자 프로필 저장, 드롭 참여 최종 판정, 외부 이메일/SMS 사업자별 프로토콜, Apple/Google 실제 로그인, passkey MVP 구현.

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) | 페이지 참조: [PAGE.A.300](../../../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md), [PAGE.A.310](../../../10-sitemap/PAGE_A_310_password_find/PAGE_A_310_password_find.md) | UI 참조: [UI.A.300](../../../20-ui/UI_A_300_auth_member/UI_A_300_auth_member.md), [UI.A.310](../../../20-ui/UI_A_310_password_find/UI_A_310_password_find.md) | UC 참조: [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) | BC 참조: [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) | 도메인 참조: [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) | 영속성 참조: [SD.A.30020](../A_300_20-persistence/README.md) | 서비스 참조: [SD.A.30030](../A_300_30-service/README.md) | 시퀀스 참조: [SCN.A.300](../../../80-sequence/A_300_auth/README.md)

## OpenAPI 계약 원장

OpenAPI 진입 문서는 HTTP 요청과 응답의 기계 판독 가능한 원장이다. 운영 코드 생성에 개발 전용 Route가 섞이지 않도록 운영 API와 개발 API의 진입 문서를 분리한다.

| 범위 | 진입 문서 | 포함 API | 코드 생성 |
| --- | --- | --- | --- |
| 운영 사용자·운영자 API | [openapi/openapi.yaml](openapi/openapi.yaml) | `API.A.300-01~29`, `API.A.300-31` | 운영 서버와 클라이언트 생성 입력 |
| 개발·테스트 전용 API | [openapi/dev.openapi.yaml](openapi/dev.openapi.yaml) | `API.A.300-30` | 운영 코드 생성에서 제외 |

- Path, method, 요청 parameter와 header, request body, 응답 schema, HTTP 상태, 오류 body와 예시는 OpenAPI를 기준으로 한다.
- Markdown은 설계 의도, 트랜잭션, 상태 변경, Context 책임, 보안 근거와 미확정 사항을 기록하며 OpenAPI 필드 표를 복제하지 않는다.
- `openapi/openapi.yaml`과 `openapi/dev.openapi.yaml`만 완전한 OpenAPI 문서다. `paths/`와 `components/`의 파일은 진입 문서에서 참조하는 조각이다.
- 각 Path Item의 `x-error-codes`는 해당 Endpoint가 공개할 수 있는 error code와 HTTP 상태의 정확한 매핑이다.
- 개별 Markdown은 아래 Endpoint 목록의 Path Item과 일대일로 연결한다.

`archive` 저장소 루트에서 다음 명령으로 참조를 포함한 계약 전체를 검증한다.

```bash
npx --yes @redocly/cli@2.38.0 lint \
  --config blueprint/50-service-design/A_300_auth/A_300_40-api/openapi/redocly.yaml \
  blueprint/50-service-design/A_300_auth/A_300_40-api/openapi/openapi.yaml \
  blueprint/50-service-design/A_300_auth/A_300_40-api/openapi/dev.openapi.yaml
```

코드 생성기는 여러 외부 조각을 직접 읽지 않고 같은 버전의 Redocly CLI로 만든 bundle을 입력으로 사용한다. 운영 API와 개발 API bundle도 분리하며 운영 코드 생성에는 운영 API bundle만 전달한다.

운영 배포 검증은 운영 bundle에 `/api/v1/dev/` 경로가 있으면 실패해야 한다. 개발 Gateway는 개발 bundle을 별도로 사용하고 `DevAccessToken`과 소유 인증 방식의 AND 조건을 적용하며, 실패 응답도 OpenAPI의 `404 ProblemDetails` 계약을 따른다.

```bash
npx --yes @redocly/cli@2.38.0 bundle \
  blueprint/50-service-design/A_300_auth/A_300_40-api/openapi/openapi.yaml \
  --output /tmp/A_300_auth.openapi.bundle.yaml

npx --yes @redocly/cli@2.38.0 bundle \
  blueprint/50-service-design/A_300_auth/A_300_40-api/openapi/dev.openapi.yaml \
  --output /tmp/A_300_auth.dev.openapi.bundle.yaml
```

## API 설계 원칙

1. 웹 로그인 상태는 서버 세션으로 관리한다. 웹 응답 본문에 access token이나 refresh token을 노출하지 않는다.
2. 모바일 앱은 짧은 수명의 access JWT와 서버 상태를 가진 opaque refresh token을 사용한다.
3. 이메일, 휴대폰 번호, 인증번호, 비밀번호, token 원문은 JWT claim, 내부 `X-User-*` 헤더, 로그, trace, Domain Event에 넣지 않는다.
4. 공개 인증 API는 계정 존재 여부를 직접 알려주지 않는다. 상세 실패 원인은 권한이 있는 운영자 Query와 감사 이벤트에서만 확인한다.
5. 사용자 프로필, 이름, 필수 동의와 추천인 정보는 프론트엔드가 Auth 검증 뒤 User 생성 API에 직접 전달한다. Auth는 이 값을 받거나 저장하지 않는다.
6. `user_id`는 Context 사용자가 발급한다. 프론트엔드가 User 생성 증거와 함께 `API.A.300-06`에 전달하며 Auth는 검증된 Identity를 해당 `user_id`에 연결한다.
7. 상태 변경 API는 명시적인 멱등성 계약을 사용한다. 인증번호 재전송처럼 사용자가 새 실행을 의도한 경우에는 새 `Idempotency-Key`를 사용한다.
8. 운영자 수동 처리는 권한, 최근 강한 인증, 승인 참조, 사유, 낙관적 동시성 검사를 모두 통과해야 한다.
9. API transaction과 integration event 발행은 `OutboxEvent`를 이용한다. 외부 소비자는 at-least-once 전달을 전제로 `event_id` 기준 멱등성을 보장한다.
10. MVP 이메일·SMS 소유 확인은 모두 사용자가 인증번호를 입력하는 code 방식을 사용한다. 링크·provider proof는 별도 계약을 추가하기 전까지 받지 않는다.
11. 전화번호 입력은 모든 Endpoint에서 `countryCode + nationalNumber` 구조를 사용하고 서버에서 E.164로 정규화한다.
12. 인증 정책은 정책별 독립 version이 아니라 하나의 활성 snapshot version과 ETag로 조회·변경한다.

## 공통 HTTP 계약

### 요청 헤더

| Header | 적용 대상 | 규칙 |
| --- | --- | --- |
| `X-Request-Id` | 전체 | 선택 입력이다. Gateway가 비어 있거나 유효하지 않으면 새 UUID를 만들고 응답에 돌려준다. |
| `traceparent` | 전체 | W3C Trace Context를 사용한다. |
| `X-Client-Channel` | `API.A.300-01` | 외부 값은 `web`, `ios`, `android` 중 하나다. Gateway가 검증하고 AuthenticationIntent에 고정한다. 이후 API는 저장된 채널을 사용한다. |
| `X-Device-Installation-Id` | 모바일 bootstrap/인증 API | 앱 설치 단위 난수 ID다. rate-limit scope에만 쓰고 사용자 프로필로 해석하지 않는다. |
| `X-Dev-Access-Token` | 개발·테스트 API | 개발 Gateway가 발급·회전하는 단기 opaque token이다. Challenge 소유 증명을 대신하지 않으며 운영 환경에는 발급자와 Route가 없다. |
| `X-App-Attestation` | 모바일 bootstrap | 운영 정책이 활성화되면 Gateway가 Apple/Google attestation을 검증한다. 원문은 Auth에 전달하지 않는다. |
| `X-Auth-Flow-Token` | 모바일 사전 인증 API | AuthenticationIntent에 묶인 단기 opaque proof다. 웹의 `__Host-dm_auth` cookie를 대신하며 Registration, VerificationChallenge, PasswordReset 요청에 사용한다. |
| `X-Registration-Status-Token` | 회원가입 상태 조회 | 회원가입 시작 응답으로 한 번 전달하는 읽기 전용 opaque proof다. terminal 상태 뒤 짧은 보존 기간까지 조회에만 쓰며 가입 완료나 Session 발급 권한은 없다. |
| `X-Refresh-Token` | 모바일 로그아웃 | 회전형 opaque refresh credential이다. URL, query, 로그에 넣지 않으며 로그아웃 request body 대신 이 header로 제출한다. `API.A.300-14` refresh는 별도 body 계약을 따른다. |
| `Idempotency-Key` | 상태 변경 API | 표에서 `필수`로 표시한 API에 UUID 형식 값을 보낸다. 같은 key와 같은 요청은 같은 논리 실행을 가리킨다. 장기 실행 API는 최종 결과 전까지 현재 상태를 반환할 수 있다. |
| `X-CSRF-Token` | 웹의 unsafe method | 사전 인증 컨텍스트 또는 로그인 세션에 묶인 CSRF token을 보낸다. 모바일 Bearer 요청에는 적용하지 않는다. |
| `Authorization: Bearer <access-jwt>` | 모바일 보호 API | 모바일 access JWT를 보낸다. 웹은 이 헤더 대신 서버 세션 cookie를 사용한다. |
| `If-Match` | 운영자 정책 변경 | 현재 활성 전체 policy snapshot ETag를 보낸다. 버전이 다르면 `412`를 반환한다. |
| `X-Audit-Reason-Code` | 운영자 인증 상태 조회 | 개인정보 없는 조회 사유 코드를 보내고 조회자와 함께 감사 기록에 남긴다. |

### 응답 Envelope

성공 응답은 데이터와 요청 식별자를 분리한다.

```json
{
  "data": {
    "example": "value"
  },
  "meta": {
    "requestId": "30d9fa85-0a18-4263-98b6-231dca5a6fb8"
  }
}
```

오류 응답은 `application/problem+json`을 사용한다.

```json
{
  "type": "https://api.dropmong.example/problems/auth-signin-failed",
  "title": "인증 요청을 완료할 수 없습니다.",
  "status": 401,
  "code": "AUTH_SIGNIN_FAILED",
  "detail": "입력한 정보를 확인한 뒤 다시 시도해주세요.",
  "retryable": false,
  "requestId": "30d9fa85-0a18-4263-98b6-231dca5a6fb8"
}
```

`detail`에는 이메일, 휴대폰 번호, 사용자 존재 여부, 내부 식별자, 잠금 정책의 내부 사유를 넣지 않는다.

### 멱등성

- 멱등성 범위는 `API ID + actor 또는 사전 인증 컨텍스트 + Idempotency-Key`다. 단, 아직 컨텍스트가 없는 `API.A.300-01`은 operation namespace 안에서 전역 UUID key를 bootstrap scope로 사용한다.
- 서버는 password, code, proof를 포함한 전체 canonical body에 전용 서버 비밀키 HMAC을 적용한 fingerprint만 `IdempotencyRecord`에 저장한다. 원문과 단순 hash는 저장하지 않는다.
- 같은 key와 같은 hash는 같은 논리 실행과 결과 참조를 재사용한다. `API.A.300-06`도 별도 작업이나 polling 없이 완료된 Session 결과를 재사용한다.
- `API.A.300-01`은 같은 key와 같은 hash에서 같은 AuthenticationIntent를 유지하되 owner proof와 웹 CSRF token을 row lock 안에서 새로 발급한다. 이전 proof는 즉시 무효화하고 proof 원문이나 bootstrap 응답 암호문은 저장하지 않는다.
- 같은 key와 다른 hash는 `409 AUTH_IDEMPOTENCY_CONFLICT`를 반환한다.
- challenge 발송 API에서 같은 key를 재전송하면 이메일/SMS를 다시 보내지 않는다.
- 비밀번호 재설정 Challenge 검증의 같은-key 모바일 성공은 같은 PasswordReset의 reset grant hash를 교체해 새 grant만 반환하고 이전 grant를 즉시 무효화한다. grant 응답 암호문은 저장하지 않는다.
- 모바일 refresh rotation은 같은 key로 재시도하면 최초 회전 결과를 돌려주고 새 token family를 추가로 만들지 않는다.
- token이나 cookie를 다시 내려줘야 하는 모바일 refresh, 이메일 재인증과 휴대폰 교체 완료의 credential 전달 복구는 짧은 TTL의 암호화된 replay payload를 별도 보안 저장소에 두고 IdempotencyRecord에는 reference만 저장한다.
- 재인증·휴대폰 교체 완료의 이전 credential은 같은 Session·operation·key·request fingerprint의 `rotated_pending_delivery` 복구 분기에서만 허용한다. 일반 인증과 다른 API에는 사용할 수 없다.
- 로그인·회원가입 완료 응답 유실은 token을 저장하지 않는다. 일반 사전 인증보다 먼저 consumed Intent의 recovery-only owner proof와 같은 Session·operation·key·request fingerprint를 확인하고, 성공 IdempotencyRecord가 가리키는 같은 Session의 SessionCredential만 안전하게 교체한다.
- 보존 기간은 해당 작업의 재시도 가능 기간보다 길어야 하며, 비밀번호나 token 평문은 IdempotencyRecord에 저장하지 않는다.

## 클라이언트별 세션 전달

### 웹

웹 로그인과 회원가입 완료 응답은 Session cookie를 설정하고 사전 인증 cookie를 같은 응답에서 폐기한다.

```http
Set-Cookie: __Host-dm_session=<opaque-random>; Path=/; HttpOnly; Secure; SameSite=Lax
Set-Cookie: __Host-dm_auth=; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=0
```

- `Domain` 속성을 사용하지 않고 `__Host-` prefix를 유지한다.
- `rememberMe=false`이면 브라우저 session cookie로 발급하고 서버 absolute TTL을 적용한다.
- `rememberMe=true`이면 정책에서 정한 `Max-Age`를 추가하되 서버 TTL보다 길게 두지 않는다.
- 로그인 성공, 권한 상승, 재인증 성공 시 session credential을 회전해 session fixation을 막는다.
- unsafe method는 `Origin` 검사와 session에 묶인 `X-CSRF-Token` 검사를 모두 통과해야 한다.
- CSRF token은 SessionCredential ID와 서버에 저장한 credential hash, CSRF key version으로 도출한다. 로그인/가입 완료 응답과 `GET /api/v1/auth/context`가 현재 값을 반환하며, 원문을 DB나 cookie에 저장하지 않는다.
- 브라우저는 CSRF token을 메모리에 두고 unsafe 요청 header에만 보낸다. Session credential이 회전되면 이전 CSRF token도 즉시 무효가 된다.
- 브라우저 JavaScript, LocalStorage, SessionStorage에 access token이나 refresh token을 저장하지 않는다.

사전 인증 단계는 별도 단기 cookie를 사용한다.

```http
Set-Cookie: __Host-dm_auth=<opaque-random>; Path=/; HttpOnly; Secure; SameSite=Lax
```

`__Host-dm_auth`는 AuthenticationIntent, Registration, PasswordReset과 CSRF token을 묶으며 로그인 성공 또는 TTL 만료 시 폐기한다.

Registration 시작 응답의 `registrationStatusToken`은 `X-Registration-Status-Token`으로만 다시 제출한다. Auth flow proof와 독립된 읽기 전용 proof이므로 로그인 성공으로 `__Host-dm_auth`나 mobile auth flow token이 폐기된 뒤에도 정책상 짧은 terminal 상태 조회 기간에는 `API.A.300-28`을 호출할 수 있다.

모바일은 `__Host-dm_auth` 대신 AuthenticationIntent 생성 응답의 `authFlowToken`을 메모리에 두고 `X-Auth-Flow-Token`으로 보낸다. `authFlowToken`은 목적, client channel, intent, 만료 시각에 묶으며 refresh token이나 로그인 credential로 사용할 수 없다.

### 모바일

모바일 로그인과 회원가입 완료 응답은 다음 값을 본문으로 반환한다.

```json
{
  "data": {
    "session": {
      "sessionId": "ses_01JXYZ...",
      "expiresAt": "2026-07-24T08:00:00Z"
    },
    "tokens": {
      "accessToken": "<signed-jwt>",
      "accessTokenExpiresAt": "2026-07-10T08:15:00Z",
      "refreshToken": "<opaque-refresh-token>",
      "refreshTokenExpiresAt": "2026-07-24T08:00:00Z"
    },
    "next": {
      "path": "/drops/drop_123",
      "intentId": "aint_01JXYZ..."
    }
  },
  "meta": {
    "requestId": "30d9fa85-0a18-4263-98b6-231dca5a6fb8"
  }
}
```

- iOS는 refresh token을 Keychain에, Android는 Keystore 기반 암호화 저장소에 저장한다.
- access JWT는 메모리에 우선 보관한다.
- refresh token은 opaque 난수이며 매 refresh마다 회전한다.
- 이전 refresh token 재사용이 감지되면 같은 family와 Session을 폐기하고 `AUTH_SESSION_REVOKED`를 반환한다.

모바일 access JWT의 최소 claim은 다음과 같다.

| Claim | 의미 |
| --- | --- |
| `iss` | DropMong 인증 발급자 |
| `sub` | Context 사용자가 발급한 `user_id` |
| `roles` | 정규화된 coarse-grained role 배열 |
| `permission_version` | 발급 시 반영한 AccessGrant version |
| `aud` | 허용된 모바일/API audience |
| `iat`, `exp` | 발급/만료 시각 |
| `jti` | token 고유 ID |
| `sid` | Session 상태 확인이 필요한 경우의 session reference |

JWT에는 `email`, `phone_number`, `identity_id`, 프로필, ACL 목록을 넣지 않는다.

### 외부 Provider token 경계

- 후속 Apple/Google OIDC를 도입하더라도 provider ID token은 callback에서 issuer, audience, nonce, signature, exp를 검증하는 1회 identity proof로만 사용한다.
- provider access token은 provider API 호출용이며 DropMong API의 Bearer credential로 받지 않는다. 브라우저·앱이 보낸 provider token을 내부 `X-User-*` context로 변환하지 않는다.
- DropMong access JWT, 내부 context JWT, provider token은 issuer, audience, signing key를 분리한다. ID token 검증 성공 뒤에는 내부 Identity/Session을 새로 발급하고 provider token 원문을 저장하지 않는다.

### 내부 사용자 컨텍스트

신뢰 경계는 외부 요청의 `X-User-*` 헤더를 제거한 뒤 검증된 세션 또는 JWT로 다음 헤더만 생성한다.

```http
X-User-Id: <user_id>
X-User-Roles: <comma-separated-canonical-roles>
X-Permission-Version: <access_grant_version>
X-Token-Id: <jwt.jti-or-session-artifact-id>
```

`X-User-Email`은 사용하지 않는다. 내부 서비스가 사용자 프로필을 필요로 하면 Context 사용자 공개 API를 별도로 호출한다.

### 개발·테스트 가상 발송

- Email과 SMS는 같은 Sender Port를 사용하고 환경에 따라 외부 Provider Adapter 또는 Virtual Adapter를 선택한다. MVP의 두 채널 모두 6자리 숫자 code를 전달한다.
- Virtual Adapter는 개발·테스트 전용 projection에 Challenge ID/version, channel, 마스킹 목적지, 만료 시각과 암호화한 code만 저장한다. 운영 migration과 운영 key ring에는 이 projection을 포함하지 않는다.
- `API.A.300-30`은 `DevAccessToken`과 원래 Challenge 소유 proof를 모두 요구한다. Challenge가 `issued`이고 projection version이 같은 경우에만 code를 반환한다.
- Challenge가 verified, failed, expired, revoked가 되면 같은 트랜잭션에서 projection ciphertext를 지운다. 정리 worker가 terminal/만료 누락분을 다시 처리한다.
- 운영 환경에서 개발 Route, Virtual Adapter, projection migration 또는 projection key가 설정되면 서비스 시작을 실패시킨다.

## Endpoint 목록

| API ID | Method / Path | 역할 | 설계 노트 | OpenAPI |
| --- | --- | --- | --- | --- |
| `API.A.300-01` | POST `/api/v1/auth/intents` | AuthenticationIntent 생성 | [Markdown](API_A_300_01_create_authentication_intent.md) | [Path Item](openapi/paths/API_A_300_01_create_authentication_intent.yaml) |
| `API.A.300-02` | GET `/api/v1/auth/methods` | 인증 수단 조회 | [Markdown](API_A_300_02_get_authentication_methods.md) | [Path Item](openapi/paths/API_A_300_02_get_authentication_methods.yaml) |
| `API.A.300-03` | POST `/api/v1/auth/registrations` | 이메일 회원가입 시작 | [Markdown](API_A_300_03_start_email_registration.md) | [Path Item](openapi/paths/API_A_300_03_start_email_registration.yaml) |
| `API.A.300-04` | POST `/api/v1/auth/registrations/{registrationId}/challenges` | 가입 Challenge 발급 | [Markdown](API_A_300_04_issue_registration_challenge.md) | [Path Item](openapi/paths/API_A_300_04_issue_registration_challenge.yaml) |
| `API.A.300-05` | POST `/api/v1/auth/registrations/{registrationId}/challenges/{challengeId}/verify` | 가입 Challenge 검증 | [Markdown](API_A_300_05_verify_registration_challenge.md) | [Path Item](openapi/paths/API_A_300_05_verify_registration_challenge.yaml) |
| `API.A.300-06` | POST `/api/v1/auth/registrations/{registrationId}/complete` | 회원가입 완료 | [Markdown](API_A_300_06_complete_registration.md) | [Path Item](openapi/paths/API_A_300_06_complete_registration.yaml) |
| `API.A.300-07` | POST `/api/v1/auth/signins/email` | 이메일 로그인 | [Markdown](API_A_300_07_sign_in_with_email.md) | [Path Item](openapi/paths/API_A_300_07_sign_in_with_email.yaml) |
| `API.A.300-08` | POST `/api/v1/auth/signins/phone/challenges` | 휴대폰 로그인 Challenge 발급 | [Markdown](API_A_300_08_issue_phone_signin_challenge.md) | [Path Item](openapi/paths/API_A_300_08_issue_phone_signin_challenge.yaml) |
| `API.A.300-09` | POST `/api/v1/auth/signins/phone/challenges/{challengeId}/verify` | 휴대폰 로그인 검증 | [Markdown](API_A_300_09_verify_phone_signin.md) | [Path Item](openapi/paths/API_A_300_09_verify_phone_signin.yaml) |
| `API.A.300-10` | POST `/api/v1/auth/password-resets` | 비밀번호 재설정 시작 | [Markdown](API_A_300_10_start_password_reset.md) | [Path Item](openapi/paths/API_A_300_10_start_password_reset.yaml) |
| `API.A.300-11` | POST `/api/v1/auth/password-resets/{passwordResetId}/challenges` | 재설정 Challenge 발급 | [Markdown](API_A_300_11_issue_password_reset_challenge.md) | [Path Item](openapi/paths/API_A_300_11_issue_password_reset_challenge.yaml) |
| `API.A.300-12` | POST `/api/v1/auth/password-resets/{passwordResetId}/challenges/{challengeId}/verify` | 재설정 Challenge 검증 | [Markdown](API_A_300_12_verify_password_reset_challenge.md) | [Path Item](openapi/paths/API_A_300_12_verify_password_reset_challenge.yaml) |
| `API.A.300-13` | PUT `/api/v1/auth/password-resets/{passwordResetId}/password` | 비밀번호 변경 | [Markdown](API_A_300_13_change_password.md) | [Path Item](openapi/paths/API_A_300_13_change_password.yaml) |
| `API.A.300-14` | POST `/api/v1/auth/sessions/refresh` | 모바일 Session 갱신 | [Markdown](API_A_300_14_refresh_session.md) | [Path Item](openapi/paths/API_A_300_14_refresh_session.yaml) |
| `API.A.300-15` | POST `/api/v1/auth/sessions/logout` | 로그아웃 | [Markdown](API_A_300_15_logout_session.md) | [Path Item](openapi/paths/API_A_300_15_logout_session.yaml) |
| `API.A.300-16` | GET `/api/v1/auth/context` | 현재 인증 컨텍스트 조회 | [Markdown](API_A_300_16_get_auth_context.md) | [Path Item](openapi/paths/API_A_300_16_get_auth_context.yaml) |
| `API.A.300-17` | POST `/api/v1/auth/reauthentications/email` | 이메일 재인증 | [Markdown](API_A_300_17_reauthenticate_email.md) | [Path Item](openapi/paths/API_A_300_17_reauthenticate_email.yaml) |
| `API.A.300-18` | POST `/api/v1/auth/method-links` | 인증 수단 연동 시작 | [Markdown](API_A_300_18_start_method_link.md) | [Path Item](openapi/paths/API_A_300_18_start_method_link.yaml) |
| `API.A.300-19` | POST `/api/v1/auth/method-links/{linkIntentId}/challenges` | 인증 수단 연동 Challenge 발급 | [Markdown](API_A_300_19_issue_method_link_challenge.md) | [Path Item](openapi/paths/API_A_300_19_issue_method_link_challenge.yaml) |
| `API.A.300-20` | POST `/api/v1/auth/method-links/{linkIntentId}/complete` | 인증 수단 연동 완료 | [Markdown](API_A_300_20_complete_method_link.md) | [Path Item](openapi/paths/API_A_300_20_complete_method_link.yaml) |
| `API.A.300-21` | POST `/api/v1/auth/phone-replacements` | 휴대폰 번호 교체 시작 | [Markdown](API_A_300_21_start_phone_replacement.md) | [Path Item](openapi/paths/API_A_300_21_start_phone_replacement.yaml) |
| `API.A.300-22` | POST `/api/v1/auth/phone-replacements/{replacementId}/challenges` | 휴대폰 번호 교체 Challenge 발급 | [Markdown](API_A_300_22_issue_phone_replacement_challenge.md) | [Path Item](openapi/paths/API_A_300_22_issue_phone_replacement_challenge.yaml) |
| `API.A.300-23` | POST `/api/v1/auth/phone-replacements/{replacementId}/complete` | 휴대폰 번호 교체 완료 | [Markdown](API_A_300_23_complete_phone_replacement.md) | [Path Item](openapi/paths/API_A_300_23_complete_phone_replacement.yaml) |
| `API.A.300-24` | GET `/api/v1/operator/auth/users/{userId}` | 운영자 인증 상태 조회 | [Markdown](API_A_300_24_get_operator_auth_user.md) | [Path Item](openapi/paths/API_A_300_24_get_operator_auth_user.yaml) |
| `API.A.300-25` | GET `/api/v1/operator/auth/policies` | 인증 정책 조회 | [Markdown](API_A_300_25_get_auth_policies.md) | [Path Item](openapi/paths/API_A_300_25_get_auth_policies.yaml) |
| `API.A.300-26` | PATCH `/api/v1/operator/auth/policies/{policyName}` | 인증 정책 변경 | [Markdown](API_A_300_26_update_auth_policy.md) | [Path Item](openapi/paths/API_A_300_26_update_auth_policy.yaml) |
| `API.A.300-27` | POST `/api/v1/operator/auth/manual-actions` | 운영자 인증 수동 처리 | [Markdown](API_A_300_27_apply_manual_auth_action.md) | [Path Item](openapi/paths/API_A_300_27_apply_manual_auth_action.yaml) |
| `API.A.300-28` | GET `/api/v1/auth/registrations/{registrationId}` | 회원가입 상태 조회 | [Markdown](API_A_300_28_get_registration_status.md) | [Path Item](openapi/paths/API_A_300_28_get_registration_status.yaml) |
| `API.A.300-29` | POST `/api/v1/auth/intents/{intentId}/action-resume` | 인증 후 행동 복구 | [Markdown](API_A_300_29_resume_authenticated_action.md) | [Path Item](openapi/paths/API_A_300_29_resume_authenticated_action.yaml) |
| `API.A.300-30` | GET `/api/v1/dev/auth/verification-messages/{challengeId}` | 가상 인증 메시지 조회 | [Markdown](API_A_300_30_get_virtual_verification_message.md) | [Path Item](openapi/paths/API_A_300_30_get_virtual_verification_message.yaml) |
| `API.A.300-31` | PUT `/api/v1/operator/auth/users/{userId}/account-status` | User 서명 proof로 계정 상태 동기 반영 | [Markdown](API_A_300_31_apply_user_account_status.md) | [Path Item](openapi/paths/API_A_300_31_apply_user_account_status.yaml) |

## 개별 API 문서

모든 Endpoint는 요청·응답·헤더·오류의 정확한 구조와 wire 예시를 OpenAPI에서 관리하고 개별 Markdown에는 책임, 트랜잭션, 멱등성, 보안 근거와 운영 판단만 둔다. 이 README는 모든 API에 적용되는 설계 원칙, 공통 HTTP 정책, 보안 정책, 시퀀스, 이벤트 계약과 운영 기준을 정의한다.

## 오류 계약

| Error Code | HTTP | 공개 조건 | 처리 |
| --- | --- | --- | --- |
| `AUTH_INPUT_INVALID` | `400` | 요청 schema 또는 형식 오류 | field 오류 목록은 PII를 포함하지 않는다. |
| `AUTH_MULTIPLE_CREDENTIALS` | `400` | 선택적 인증 API에 웹 cookie와 모바일 Bearer를 동시에 제출 | 한 채널의 credential만 남겨 다시 요청한다. |
| `AUTH_REGISTRATION_NOT_FOUND` | `404` | Registration 부재, 소유 증명 누락·오류 또는 binding 불일치 | 리소스 존재 여부를 더 공개하지 않는다. |
| `AUTH_INTENT_NOT_FOUND` | `404` | AuthenticationIntent 부재 또는 현재 사전 인증·Session binding 불일치 | Intent 존재 여부와 다른 사용자의 행동 정보를 숨긴다. |
| `AUTH_OPERATOR_TARGET_NOT_FOUND` | `404` | 운영자 권한 검증 뒤에도 조회·처리 가능한 대상이 없음 | 대상 종류와 내부 삭제 상태를 추가로 공개하지 않는다. |
| `AUTH_REDIRECT_INVALID` | `400` | 허용되지 않은 복귀 위치 | 홈 또는 안전한 기본 경로로 다시 시작한다. |
| `AUTH_INTENT_EXPIRED` | `410` | AuthenticationIntent 만료 | 새 intent를 만든다. |
| `AUTH_REGISTRATION_EXPIRED` | `410` | 소유 확인된 Registration 완료 기한 만료 | 새 Registration을 시작한다. |
| `AUTH_METHOD_UNAVAILABLE` | `409` | 비활성 인증 수단 | 활성 수단 선택으로 돌아간다. |
| `AUTH_IDEMPOTENCY_CONFLICT` | `409` | 같은 key에 다른 payload 또는 진행 중인 Registration에 다른 완료 key 사용 | 일반 명령은 새 key로 다시 실행할 수 있지만, 진행 중인 회원가입 완료는 최초 key를 사용한다. |
| `AUTH_IDENTIFIER_UNAVAILABLE` | `409` | 가입 식별자를 사용할 수 없음 | 중복, 예약, 위험 판정을 구분하지 않는 안내를 사용한다. |
| `AUTH_USER_CREATION_PROOF_INVALID` | `403` | User 생성 증거 무효·만료·registration 또는 user 불일치 | 프론트엔드가 같은 registration으로 User 결과를 다시 확인한다. |
| `AUTH_CHALLENGE_FAILED` | `400` | code 불일치 또는 검증 실패 | 남은 횟수 대신 일반 안내를 기본값으로 둔다. |
| `AUTH_CHALLENGE_EXPIRED` | `410` | challenge TTL 만료 | 새 challenge를 발급한다. |
| `AUTH_PASSWORD_RESET_GRANT_EXPIRED` | `410` | 검증된 비밀번호 재설정 권한 또는 모바일 grant 만료 | 비밀번호 재설정을 처음부터 다시 진행한다. |
| `AUTH_VERIFICATION_REQUIRED` | `409` | 회원가입 완료에 필요한 이메일·휴대폰 소유 확인 미완료 | 누락된 필수 인증을 완료한다. |
| `AUTH_VIRTUAL_MESSAGE_NOT_FOUND` | `404` | 개발·테스트 가상 메시지 부재, 외부 delivery mode 또는 Challenge 소유 binding 불일치 | 리소스 존재 여부를 더 공개하지 않는다. |
| `AUTH_VIRTUAL_MESSAGE_UNAVAILABLE` | `410` | 가상 메시지 검증·만료·실패·폐기 또는 code 원문 폐기 완료 | 새 Challenge를 발급한다. |
| `AUTH_RATE_LIMITED` | `429` | 발급, 검증, 로그인 제한 초과 | `Retry-After`를 반환한다. |
| `AUTH_SIGNIN_FAILED` | `401` | 이메일/비밀번호 검증 실패 | 계정 존재 여부를 숨긴다. |
| `AUTH_ACCOUNT_LOCKED` | `423` | 올바른 credential 확인 후 잠금 상태 | `unlockAvailableAt`을 반환할 수 있다. |
| `AUTH_PASSWORD_RESET_REQUIRED` | `403` | 올바른 이메일 credential 확인 후 강제 재설정 상태 | 비밀번호 재설정 API로 이동한다. |
| `AUTH_USER_RESTRICTED` | `403` | credential/OTP 확인 뒤 사용자 제한·비활성 상태 | 새 Session/refresh/재인증을 거부하고 지원 안내를 제공한다. |
| `AUTH_PHONE_IDENTITY_NOT_LINKED` | `409` | SMS 소유 확인 후 active Link 없음 | 회원가입 또는 로그인 후 연동을 안내한다. |
| `AUTH_SESSION_REQUIRED` | `401` | 보호 API에 인증 정보 없음 | 로그인 intent를 만든다. |
| `AUTH_SESSION_REVOKED` | `401` | Session 폐기 또는 refresh 재사용 탐지 | 로컬 credential을 폐기하고 다시 로그인한다. |
| `AUTH_SESSION_DELIVERY_EXPIRED` | `410` | 회원가입 자동 로그인 발급 기한 또는 로그인·회원가입·휴대폰 교체 credential 전달 복구 기간 종료 | 이미 확정된 IdentityLink는 유지하고 새 로그인·재인증 action으로 인증한다. |
| `AUTH_REFRESH_RETRY_EXPIRED` | `410` | 같은-key refresh 암호화 응답의 복구 TTL 종료 | token을 다시 보내지 말고 새로 로그인한다. |
| `AUTH_FORBIDDEN` | `403` | role 또는 permission 부족 | 리소스 존재 여부를 추가로 노출하지 않는다. |
| `AUTH_CSRF_INVALID` | `403` | 웹 CSRF/Origin 검증 실패 | 새 인증 컨텍스트를 시작한다. |
| `AUTH_REAUTH_REQUIRED` | `403` | 고위험 작업에 recent auth 없음 | 재인증 endpoint로 이동한다. |
| `AUTH_REAUTHENTICATION_PROOF_INVALID` | `410` | ReauthenticationProof 만료·소비·목적 불일치 | 목적에 맞게 재인증한다. |
| `AUTH_IDENTITY_LINK_CONFLICT` | `409` | 이미 다른 사용자에 active 연결 | 계정 병합 없이 수동 처리 안내를 제공한다. |
| `AUTH_IDENTITY_LINK_NOT_FOUND` | `404` | requested IdentityLink 부재 또는 현재 Session binding 불일치 | 리소스 존재 여부를 더 공개하지 않는다. |
| `AUTH_IDENTITY_LINK_INTENT_EXPIRED` | `410` | 인증 수단 연동·휴대폰 교체 requested Link 완료 기한 만료 | 새 연동 또는 교체 요청을 시작한다. |
| `AUTH_PASSWORD_POLICY_NOT_MET` | `422` | 새 비밀번호 정책 미충족 | 공개 가능한 정책 항목만 반환한다. |
| `AUTH_PASSWORD_RESET_INVALID` | `400` | reset intent/grant 검증 실패 | 세부 원인을 합치고 처음부터 다시 시작한다. |
| `AUTH_POLICY_PRECONDITION_FAILED` | `412` | `If-Match` policy version 불일치 | 최신 정책을 다시 조회한다. |
| `AUTH_RESOURCE_PRECONDITION_FAILED` | `412` | 운영자 수동 처리 대상 version 불일치 | 최신 대상 상태를 다시 조회하고 승인 binding을 확인한다. |
| `AUTH_APPROVAL_REQUIRED` | `409` | 승인 누락/만료/대상 불일치 | 유효한 승인 참조를 받는다. |
| `AUTH_SERVICE_UNAVAILABLE` | `503` | 인증 저장소/필수 의존성 장애 | `Retry-After`와 안정된 일반 메시지를 반환한다. |

CS/운영자 Query에는 별도 내부 reason code를 제공할 수 있지만 공개 오류 code를 상세 내부 사유와 일대일로 매핑하지 않는다.

## 보안 정책

### 계정 존재 여부 보호

- 이메일 로그인 실패는 존재하지 않는 계정, 비밀번호 오류, 비활성 Identity를 `AUTH_SIGNIN_FAILED`로 합친다.
- 비밀번호 재설정 시작과 challenge 발급은 계정/수단 존재 여부와 관계없이 `202`를 반환한다.
- 휴대폰 로그인 challenge 시작은 IdentityLink 존재 여부를 알리지 않는다.
- 휴대폰 번호 미연결은 해당 번호의 SMS challenge를 통과한 사용자에게만 알려준다.
- 가입 식별자 중복은 `AUTH_IDENTIFIER_UNAVAILABLE`로 일반화하고 `user_id`나 연결 수단을 노출하지 않는다.
- 응답 시간 차이를 줄이고 고비용 password hash 검증에는 적절한 dummy verification을 사용한다.

### 초기 제한값

아래 값은 첫 운영 기본값이며 배포 없이 정책으로 변경할 수 있어야 한다.

| 대상 | 초기 제한 | 기준 key |
| --- | --- | --- |
| AuthenticationIntent 생성 | 분당 30회 | IP + 사전 인증 컨텍스트 |
| Registration 생성 | 시간당 5회 | IP + normalized identifier |
| 이메일/SMS challenge 발급 | 15분당 3회, 하루 10회 | purpose + destination + IP |
| challenge code 검증 | challenge당 5회 | challenge ID |
| 이메일 로그인 실패 | 15분 동안 5회 후 30분 잠금 | Identity + IP 보조 제한 |
| 휴대폰 로그인 시작 | 15분당 3회 | phone blind index + IP |
| 비밀번호 재설정 시작 | 15분당 3회 | identifier blind index + IP |
| 모바일 refresh | 분당 30회 | Session + refresh family |
| 운영자 수동 처리 | 분당 10회 | 운영자 `user_id` |

- IP만으로 사용자 계정을 잠그지 않는다.
- 성공 로그인 시 실패 count 초기화 기준과 잠금 만료 처리는 LoginLockPolicy에서 정한다.
- 제한 응답에는 `Retry-After`를 포함하되 내부 threshold 전체를 노출하지 않는다.
- 테스트 번호 예외는 production에서 비활성화하고 별도 환경 설정과 감사 대상으로 둔다.

### CSRF와 redirect

- 웹 `POST /auth/intents`는 아직 CSRF token이 없으므로 `Origin` allowlist와 Fetch Metadata로 bootstrap 요청을 제한한다.
- 모바일 bootstrap은 브라우저 Origin/Fetch Metadata를 요구하지 않는다. Gateway가 `ios|android` 채널, 설치 ID 형식, rate limit, 활성화된 app-attestation 정책을 검증한 뒤 Auth에 전달한다.
- 이후 웹 unsafe method는 `Origin` allowlist, `X-CSRF-Token`, `__Host-dm_auth` 또는 `__Host-dm_session` 바인딩을 검증한다.
- 로그인 성공 후 이동할 값은 AuthenticationIntent의 server-side `returnPath`만 사용한다.
- 요청 body나 query의 즉석 `redirectUrl`로 이동하지 않는다.
- AuthenticationIntent는 단기 TTL과 1회 소비를 기본으로 하며 로그인 성공 시 consumed 상태로 전환한다.

### 민감 정보 처리

- 비밀번호와 code는 요청 body에서만 받고 구조화 로그, trace attribute, metric label에 넣지 않는다.
- refresh token, reset grant, ReauthenticationProof 원문은 hash 또는 서버측 reference로 검증한다. role/permission AccessGrant는 token secret이 아니라 versioned 권한 snapshot이다.
- 이메일과 휴대폰 번호는 정규화 조회 키와 암호화 표시값을 분리하고 응답에서는 마스킹한다.
- 운영자 Query도 기본적으로 마스킹하며 unmask API는 이 설계에 포함하지 않는다.
- 요청/응답 body 샘플링은 인증 endpoint에서 기본 비활성화한다.

## 관련 처리 시퀀스

| Scenario ID | 처리 시퀀스 | 관련 API |
| --- | --- | --- |
| [`SCN.A.300-01`](../../../80-sequence/A_300_auth/SCN_A_300_01_email_registration.md) | 이메일 회원가입과 자동 로그인 | `API.A.300-03~06`, `API.A.300-28` |
| [`SCN.A.01-04`](../../A_01_user/A_01_50-sequence/SCN_A_01_04_change_user_status.md) | 사용자 계정 상태 변경과 Session 폐기 | `API.A.300-31` |
| [`SCN.A.300-02`](../../../80-sequence/A_300_auth/SCN_A_300_02_phone_signin_refresh.md) | 휴대폰 로그인과 refresh token 회전 | `API.A.300-08`, `API.A.300-09`, `API.A.300-14` |
| [`SCN.A.300-03`](../../../80-sequence/A_300_auth/SCN_A_300_03_phone_replacement.md) | 휴대폰 번호 교체 | `API.A.300-17`, `API.A.300-21~23` |
| [`SCN.A.310-01`](../../../80-sequence/A_300_auth/SCN_A_310_01_password_reset.md) | 비밀번호 재설정 | `API.A.300-10~13` |

여러 Endpoint 또는 Context를 연결하는 Mermaid 다이어그램은 위 시퀀스 문서를 단일 기준으로 사용한다. 단일 Endpoint 내부 처리만 설명하는 다이어그램은 해당 API 문서에 둘 수 있다.

## 도메인 매핑

| API 묶음 | Application Command / Query | Aggregate / Entity | Domain Event / Integration Event |
| --- | --- | --- | --- |
| `API.A.300-01~02` | 로그인 의도 보존, 인증 수단 조회 | AuthenticationIntent | `EVT.A.300-02 로그인 의도 보존됨` |
| `API.A.300-03~06`, `API.A.300-28` | 가입 시작, challenge 발급/검증, User 증거 확인, Session 완료/상태 조회 | Registration, VerificationChallenge, Identity, IdentityLink, PasswordCredential, Session | 가입 핵심 처리에는 Integration Event 없음 |
| `API.A.300-07` | `CMD.A.300-07 이메일 로그인`, 세션 발급 | Identity, PasswordCredential, Session | 성공: `EVT.A.300-06`, `EVT.A.300-07`; 실패/잠금: `EVT.A.300-31`, `EVT.A.300-32` |
| `API.A.300-08~09` | `CMD.A.300-05 휴대폰 소유 확인`, `CMD.A.300-08 휴대폰 번호 로그인` | VerificationChallenge, Identity, IdentityLink, Session | `EVT.A.300-05`, `EVT.A.300-06`, `EVT.A.300-07`, `EVT.A.300-33` |
| `API.A.300-10~13` | 비밀번호 재설정 시작, challenge 검증, 비밀번호 변경 | PasswordReset, VerificationChallenge, PasswordCredential, Session | `EVT.A.300-22`, `EVT.A.300-25`, `EVT.A.300-26`, `EVT.A.300-33` |
| `API.A.300-14` | `CMD.A.300-10 토큰 재발급` | Session, SessionCredential | `EVT.A.300-08 토큰 재발급됨`, `EVT.A.300-27` |
| `API.A.300-15` | `CMD.A.300-11 로그아웃` | Session, SessionCredential | `EVT.A.300-09 사용자 로그아웃됨` |
| `API.A.300-16` | 현재 인증 컨텍스트 조회 | Session read model, AccessGrant | event 없음 |
| `API.A.300-17` | `CMD.A.300-15 이메일 재인증` | Session, PasswordCredential, ReauthenticationProof | Domain Event 없음, 보안 감사 integration event |
| `API.A.300-18~20` | `CMD.A.300-12 인증 수단 연동 요청`, `CMD.A.300-17~18` | VerificationChallenge, Identity, IdentityLink, ReauthenticationProof | `EVT.A.300-10`, `EVT.A.300-03`, `EVT.A.300-33` |
| `API.A.300-21~23` | `CMD.A.300-14 휴대폰 번호 셀프 변경`, `CMD.A.300-17~18` | VerificationChallenge, Identity, IdentityLink, ReauthenticationProof, Session | `EVT.A.300-18~20`, `EVT.A.300-26`, `EVT.A.300-33` |
| `API.A.300-24` | 인증 상태 운영 Query | UserAuthState, Identity, IdentityLink, Session read model | event 없음 |
| `API.A.300-25~26` | 활성 인증 정책 snapshot 조회, 새 전체 snapshot 생성 | AuthenticationPolicySnapshot | `EVT.A.300-30 인증 정책 변경됨` |
| `API.A.300-27` | `CMD.A.300-13 인증 수단 수동 처리` 동기 실행 | Identity, IdentityLink, Session | `EVT.A.300-11` 및 action별 감사 event |
| `API.A.300-29` | `CMD.A.300-24 인증 후 행동 복구` | AuthenticationIntent, ActionIntentPayload | Domain Event 없음, 전달 감사 integration event |
| `API.A.300-30` | 가상 인증 메시지 조회 Query | VerificationChallenge, VirtualVerificationMessage read model | Domain Event 없음, code 없는 조회 감사 event |
| `API.A.300-31` | `CMD.A.300-26 사용자 계정 상태 반영` | UserAuthState, Session | Integration Event 없음 |

API는 [도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)의 Event ID를 사용한다. event 전송 성공 여부를 business event 이름으로 사용하지 않고 실제 업무 사실을 OutboxEvent에 기록한다.

## 가입 증거 계약

### registrationCompletionProof

Auth가 두 필수 Challenge 완료 뒤 프론트엔드에 반환하는 짧은 수명의 서명 증거다. `registrationId`, `purpose=create_user`, `audience=user-service`, 검증 완료 시각, 만료 시각, nonce와 key ID만 포함한다. 이메일·휴대폰·인증번호 원문은 포함하지 않는다.

### userCreationProof

User가 생성 성공 뒤 프론트엔드에 반환하고 Auth 완료 API가 검증하는 짧은 수명의 서명 증거다. `registrationId`, `userId`, `audience=auth-service`, 생성 시각, 만료 시각과 key ID를 포함한다.

가입 완료는 프론트엔드가 두 증거를 각 대상 API에 직접 전달해 처리한다. 가입 완료를 진행시키는 Integration Event는 없다.

## Integration Event 계약

### 공통 envelope

```json
{
  "eventId": "evt_01JXYZ...",
  "eventType": "Auth.IdentityLinkReplaced",
  "eventVersion": 1,
  "occurredAt": "2026-07-10T08:00:00Z",
  "aggregateId": "ilnk_01JXYZ...",
  "correlationId": "30d9fa85-0a18-4263-98b6-231dca5a6fb8",
  "causationId": "cmd_01JXYZ...",
  "userId": "usr_01JXYZ...",
  "data": {
    "identityType": "phone",
    "reason": "phone_change"
  }
}
```

- `eventId`는 전역 unique이고 소비자 멱등성 key다.
- `eventType + eventVersion`이 schema 식별자다.
- `data`에는 이메일, 휴대폰 번호, 비밀번호, code, token, provider secret, 증빙 원문을 넣지 않는다.
- Session 관련 event는 `sessionId`와 상태만 전달하고 credential 원문을 전달하지 않는다.
- Audit Context 전달은 transactional outbox와 재시도를 사용하며 API transaction 안에서 외부 broker 응답을 기다리지 않는다.
- 실패한 outbox 전달은 지표와 운영 경보로 드러내되 API 성공을 감사 저장소 동기 응답에 의존시키지 않는다.

## 관측성과 운영

### 구조화 로그

허용 필드:

- `request_id`, `trace_id`, `api_id`, `http_status`, `duration_ms`
- `client_channel`, `auth_method`, `outcome`, 공개 `error_code`
- opaque `user_id`, `session_id`, `registration_id`, `challenge_id`
- `idempotency_replayed`, `rate_limit_policy`, `outbox_event_id`

금지 필드:

- 이메일, 휴대폰 번호, 이름, 주소, 추천인 코드
- 비밀번호, 인증번호, access/refresh/reset/access grant 원문
- cookie, Authorization header, 전체 요청/응답 body

### 메트릭

| Metric | Label |
| --- | --- |
| `auth_api_requests_total` | `api_id`, `status_class`, `outcome` |
| `auth_api_duration_seconds` | `api_id`, `client_channel` |
| `auth_signin_attempts_total` | `method`, `outcome` |
| `auth_challenge_issued_total` | `purpose`, `method`, `outcome` |
| `auth_challenge_verified_total` | `purpose`, `outcome` |
| `auth_registration_completed_total` | `client_channel`, `outcome` |
| `auth_registration_link_wait_seconds` | `outcome` |
| `auth_session_refresh_total` | `outcome` |
| `auth_refresh_reuse_detected_total` | `client_channel` |
| `auth_identity_lock_total` | `method` |
| `auth_manual_action_total` | `action`, `outcome` |
| `auth_outbox_pending` | `event_type` |
| `auth_inbox_messages` | `message_type`, `outcome` |

identifier, `user_id`, `session_id`는 metric label로 사용하지 않는다.

### Trace

- Controller, Application Service, Repository, User proof 검증과 Session 저장을 별도 span으로 만든다.
- 이메일/SMS provider span에는 provider operation과 결과만 기록하고 destination을 attribute로 넣지 않는다.
- 비밀번호 hash와 token 검증 구간은 span 이름만 남기며 입력값을 기록하지 않는다.
- User API 오류, 프론트엔드 재시도와 Auth transaction 실패를 구분해 가입 실패 원인을 확인할 수 있어야 한다.

### 장애 기준

- 인증 저장소 장애 시 신규 로그인, refresh, logout, 연동 변경은 `503`으로 실패시킨다.
- 이미 검증 가능한 모바일 access JWT는 만료 전까지 Gateway 정책으로 처리할 수 있지만 고위험 API는 최신 Session 확인 실패 시 거부한다.
- 공개 페이지의 선택적 개인화 인증 실패는 해당 컴포넌트만 비회원/일시 실패 상태로 바꾸고 공개 데이터 응답은 유지한다.
- 이메일/SMS provider 장애는 challenge 상태를 성공으로 만들지 않고 `AUTH_SERVICE_UNAVAILABLE` 또는 발송 전용 오류로 반환한다.
- User 일시 장애는 동기 오류로 반환하고 같은 registration과 key로 재시도한다. Auth는 `user_id`를 임의 생성하지 않는다.

## Versioning

- REST URI major version은 `/api/v1`로 관리한다.
- 기존 JSON field 추가는 optional additive change로 처리한다.
- enum을 받는 클라이언트는 알 수 없는 값을 안전한 기본 상태로 처리해야 한다.
- field 삭제, 의미 변경, 인증 방식 변경은 새 major version 또는 명시적 migration 기간을 둔다.
- JWT claim과 REST response schema는 별도 계약이다. REST version 변경이 JWT claim 확장을 자동으로 의미하지 않는다.
- Integration Event는 `eventVersion`을 올리고 producer가 전환 기간 동안 필요한 호환 schema를 제공한다.
- 운영자 정책 Query는 활성 전체 snapshot의 ETag를 반환하고 변경 API는 같은 ETag의 `If-Match`로 동시 변경 충돌을 막는다.
- 폐기 예정 endpoint는 `Deprecation`, `Sunset`, `Link` header와 운영 공지를 함께 제공한다.

## 검증 시나리오

| 시나리오 | 기대 결과 |
| --- | --- |
| 외부 URL을 returnPath로 전달 | `400 AUTH_REDIRECT_INVALID`, AuthenticationIntent 미생성 |
| 같은 가입 key와 같은 body 재요청 | 같은 Registration과 현재 상태 반환, IdentityLink·Session 중복 생성 없음 |
| 같은 완료 key와 User proof 재요청 | 기존 논리 Session 결과 재사용 |
| 같은 registration에 다른 `user_id` 전달 | `AUTH_IDEMPOTENCY_CONFLICT`, 재귀속·병합 없음 |
| 같은 가입 key에 다른 body 요청 | `409 AUTH_IDEMPOTENCY_CONFLICT` |
| 등록되지 않은 이메일로 재설정 시작 | 등록 이메일과 같은 `202` 응답 형식, 실제 발송 없음 |
| 휴대폰 challenge 성공 후 IdentityLink 없음 | `409 AUTH_PHONE_IDENTITY_NOT_LINKED`, 사용자 자동 생성 없음 |
| 웹 로그인 성공 | HttpOnly cookie 발급, body에 access/refresh token 없음 |
| 모바일 로그인 성공 | 최소 claim access JWT와 opaque refresh token 반환 |
| refresh 재시도에 같은 key 사용 | 최초 회전 결과 재사용, 추가 token 생성 없음 |
| rotated refresh token을 새 key로 재사용 | family와 Session 폐기, 재사용 탐지 event 발행 |
| 비밀번호 재설정 완료 | PasswordCredential 교체, 기존 Session 전체 폐기, 자동 로그인 없음 |
| 휴대폰 교체 동시 요청 | 하나만 성공, active phone Link 하나 유지 |
| 외부에서 `X-User-Email` 전달 | Gateway가 제거하고 upstream에 전달하지 않음 |
| CS 권한으로 정책 변경 시도 | `403 AUTH_FORBIDDEN` |
| 오래된 ETag로 정책 변경 | `412 AUTH_POLICY_PRECONDITION_FAILED` |
| 승인 없는 수동 재연동 | `409 AUTH_APPROVAL_REQUIRED` |
| 오래된 대상 version으로 수동 처리 | `412 AUTH_RESOURCE_PRECONDITION_FAILED`, 업무 변경과 감사 event 미생성 |
| 완료·실패·만료 Registration 상태 조회 | 소유 증명이 유효하면 terminal 상태를 `200` body로 반환하고 Session을 발급하지 않음 |
| 소유 컨텍스트가 다른 가상 메시지 조회 | `404 AUTH_VIRTUAL_MESSAGE_NOT_FOUND`, Challenge와 Identity 정보 노출 없음 |
| 운영 환경에서 개발·테스트 API 호출 | Route 미등록으로 `404`, virtual delivery 설정이면 서비스 시작 실패 |

## 확인 필요

- AuthenticationIntent와 Registration, PasswordReset의 정확한 TTL을 확정한다.
- 웹 session idle TTL과 absolute TTL, `rememberMe`별 값을 확정한다.
- ReauthenticationProof 목적 목록과 최근 인증 허용 시간을 확정한다.
- 운영자 수동 처리 approval provider와 `evidenceRef` 접근 권한을 확정한다.
- 웹 server session을 Auth에서 직접 검증할지 Ingress의 Auth 연동으로 signed request context를 만들지 배포 설계에서 확정한다. 어느 경우에도 Ingress는 업무 응답을 조합하지 않는다.
- 운영자 정책 변경에 허용할 최소/최대 값과 승인 단계를 확정한다.
- 가상 인증 메시지 전달과 조회를 설명하는 `SCN.A.300-04` 시퀀스 문서를 추가한다.
