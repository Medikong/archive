---
id: API.A.300-06
title: 회원가입 완료 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, registration, session]
source: local
created: 2026-07-10
updated: 2026-07-13
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
| 역할 | 프론트엔드가 생성한 User를 Registration에 연결하고 Session을 발급한다. |
| API 유형 | 동기 Command |
| 인증 | 가입 소유 proof + User 생성 proof |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |

## 요청

```json
{
  "userId": "00000000-0000-0000-0000-000000000001",
  "userCreationProof": "opaque-signed-proof"
}
```

## 성공 응답

`200 OK`. 웹은 HttpOnly Session cookie를 설정하고 모바일은 Session credential을 본문으로 반환한다.

```json
{
  "data": {
    "registrationId": "reg_01JXYZ...",
    "userId": "00000000-0000-0000-0000-000000000001",
    "status": "completed",
    "next": "/"
  }
}
```

## 처리 규칙

1. Registration 소유 proof, 두 필수 Challenge의 verified 상태와 만료를 확인한다.
2. User 공개 키로 `userCreationProof`의 registration ID, user ID, 발급·만료와 서명을 검증한다.
3. 두 IdentityLink, Session, SessionCredential, AccessGrant와 Registration `completed`를 한 트랜잭션에 저장한다.
4. 커밋 뒤 채널에 맞는 Session credential을 반환한다.

Auth는 User를 생성하거나 User API를 호출하지 않는다. 프론트엔드가 User 생성 성공 뒤 이 API를 호출한다.

## 멱등성과 실패 처리

- 같은 registration, user ID, 요청 fingerprint와 key는 같은 논리 Session 결과를 반환한다.
- 같은 registration에 다른 user ID 또는 proof가 오면 `AUTH_IDEMPOTENCY_CONFLICT`로 거부한다.
- 트랜잭션 실패 시 IdentityLink와 Session을 모두 롤백한다.
- 응답 유실 시 같은 key 재시도로 기존 논리 Session의 전달만 복구한다.

## 오류

| Code | HTTP | 조건 |
| --- | --- | --- |
| `AUTH_VERIFICATION_REQUIRED` | 409 | 필수 Challenge 미완료 |
| `AUTH_REGISTRATION_EXPIRED` | 410 | Registration 또는 proof 만료 |
| `AUTH_USER_CREATION_PROOF_INVALID` | 403 | User 증거 무효·registration/user 불일치 |
| `AUTH_IDEMPOTENCY_CONFLICT` | 409 | 같은 key의 다른 요청 |
| `AUTH_DEPENDENCY_UNAVAILABLE` | 503 | AccessGrant 등 필수 의존성 장애 |

## 연관 문서

- [Auth 가입 시퀀스](../../../80-sequence/A_300_auth/SCN_A_300_01_email_registration.md)
- [User 생성 API](../../A_01_user/A_01_40-api/API_A_01_01_create_user.md)

## 검증 항목

- User proof가 없거나 무효면 IdentityLink와 Session을 만들지 않는다.
- IdentityLink, Session과 Registration 완료가 한 트랜잭션에 저장된다.
- 같은 요청 재시도에서 논리 Session이 하나만 존재한다.
- Event Broker와 상태 polling 없이 동기 응답한다.
