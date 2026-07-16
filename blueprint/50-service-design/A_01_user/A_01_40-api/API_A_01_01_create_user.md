---
id: API.A.01-01
title: User 생성 API
type: service-design-api-endpoint
status: draft
tags: [service-design, user, api, registration]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
api_design: SD.A.0140
domain_model: SD.A.0110
---

# API.A.01-01 User 생성

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/users` |
| 역할 | Auth 검증을 마친 가입 요청으로 User를 생성한다. |
| 인증 | Auth가 발급한 `create_user` 전용 `registrationCompletionProof` |
| 멱등성 | `registrationId` 전역 유일 |

## 요청

```json
{
  "registrationId": "reg_01JXYZ...",
  "registrationCompletionProof": "opaque-signed-proof",
  "profile": {
    "privateName": "홍길동",
    "nickname": "dropfan",
    "introduction": null
  },
  "requiredAgreements": [
    {
      "agreementCode": "TERMS_OF_SERVICE",
      "agreementVersion": "2026-07-01",
      "acceptedAt": "2026-07-13T10:00:00Z"
    }
  ]
}
```

## 성공 응답

`201 Created`. 같은 `registrationId` 재요청은 `200 OK`로 같은 논리 결과를 반환할 수 있다.

```json
{
  "data": {
    "userId": "00000000-0000-0000-0000-000000000001",
    "userVersion": 1,
    "createdAt": "2026-07-13T10:00:01Z",
    "userCreationProof": "opaque-signed-proof"
  }
}
```

## 처리 규칙

- Auth proof의 registration, 검증 완료, 만료와 서명을 확인한다.
- proof의 purpose가 `create_user`, audience가 User 서비스인지 확인한다.
- 현재 필수 동의 code와 version을 모두 요구한다.
- User, 동의 이력과 멱등 결과를 한 트랜잭션에 저장한다.
- Auth, Agreement와 Event Broker를 호출하지 않는다.
- 프론트엔드는 반환된 `userCreationProof`를 Auth 가입 완료 API에 바로 전달하고 영구 저장하지 않는다.
- proof는 URL, 브라우저 영구 저장소, 로그와 trace에 넣지 않으며 웹 요청은 허용된 Origin에서만 받는다.

## 오류

- `403 USER_REGISTRATION_PROOF_INVALID`
- `409 USER_REGISTRATION_CONFLICT`
- `422 USER_REQUIRED_AGREEMENT_INVALID`

## 연관 시퀀스

- [SCN.A.01-01 회원가입과 자동 로그인](../../../80-sequence/A_01_user/SCN_A_01_01_user_provisioning_auth_link.md)
