---
id: API.A.300-31
title: 사용자 계정 상태 반영 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
created: 2026-07-13
updated: 2026-07-16
---

# API.A.300-31 사용자 계정 상태 반영

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `PUT /api/v1/operator/auth/users/{userId}/account-status` |
| operationId | `applyUserAccountStatus` |
| 역할 | 운영 프론트엔드가 User의 서명 proof를 Auth에 동기 반영한다. |
| API 유형 | Command |
| 인증 | `Authorization: Bearer <access-jwt>`, 최근 strong auth |
| 외부 인가 | 검증된 `user.account_status.change` 결정 |
| 멱등성 | `statusChangeId`와 `userVersion` |
| HTTP 응답 캐시 | `no-store` |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_31_apply_user_account_status.yaml](openapi/paths/API_A_300_31_apply_user_account_status.yaml)

## 요청과 응답

운영 프론트엔드는 `API.A.01-07`이 반환한 proof를 수정하지 않고 전달한다.

```json
{
  "userStatusChangeProof": "opaque-user-signed-proof"
}
```

proof는 `userId`, `statusChangeId`, `accountStatus`, `userVersion`, `changedAt`, `audience=auth-service`, 발급·만료 시각과 nonce를 User 서명으로 보호한다. 응답은 Auth가 반영한 현재 상태와 version, 실제 적용 여부를 반환한다. 더 낮은 `userVersion`은 현재 상태를 바꾸지 않고 `applied=false`로 응답한다.

Auth는 이 처리에서 운영자의 role, permission, 판매자 소속 또는 업무 ACL을 조회하거나 저장하지 않는다. 외부 인가 경계가 action 단위 결정 proof를 만들고 Istio 연동 계층이 위조된 헤더를 차단한다.

## 처리 규칙

1. access JWT의 Session, 외부 `user.account_status.change` 결정과 최근 strong auth를 확인한다.
2. User 공개 키로 proof의 서명·audience·만료를 확인하고 path `userId`와 proof의 `userId`가 같은지 검증한다.
3. `UserAuthState`를 잠그고 저장된 `userVersion`과 proof의 version을 비교한다.
4. 더 높은 version이면 상태를 갱신한다.
5. `restricted` 또는 `deactivated`이면 같은 트랜잭션에서 해당 사용자의 활성 Session을 폐기한다.
6. `active` 복귀는 새 Session을 만들거나 과거 Session을 되살리지 않는다.

같은 `userVersion`과 같은 `statusChangeId`는 기존 결과를 반환한다. 같은 version에 다른 상태나 변경 ID가 들어오면 `412 AUTH_RESOURCE_PRECONDITION_FAILED`로 거부한다. 이 처리는 Broker나 상태 변경 Integration Event를 사용하지 않는다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| 운영자 `SessionStatusProjection` | 사용 | 계정 상태를 바꾸는 운영자의 Session 상태를 먼저 확인한다. 권한 판정은 상위 운영자 인가 계층이 담당한다. |
| `UserAuthState`, version, `Session` | 우회 | 계정 상태 전이, 낙관적 잠금과 영향받는 Session 선정을 PostgreSQL 트랜잭션으로 확정한다. |
| 대상 사용자의 `SessionStatusProjection` | 무효화 | `restricted` 또는 `deactivated`로 바뀌면 모든 영향받는 Session을 커밋 뒤 `revoked` 상태로 기록한다. |
| `active` 전환 | 사용하지 않음 | 계정 상태를 다시 활성화해도 이미 폐기된 Session을 되살리지 않으며 새 로그인을 요구한다. |

## 실패와 재시도

- Auth 일시 장애이면 운영 프론트엔드가 동일한 proof로 다시 호출한다.
- User 상태는 이미 확정됐으므로 Auth 실패 때문에 되돌리지 않는다.
- 더 낮은 version의 늦은 요청은 성공 형태의 no-op으로 끝낸다.
- 상태 반영과 Session 폐기 중 하나라도 실패하면 Auth 트랜잭션 전체를 롤백한다.
- proof가 변조·만료되었거나 path 사용자와 다르면 `AUTH_USER_STATUS_PROOF_INVALID`로 거부한다.

## 연관 문서

- [사용자 계정 상태 변경 시퀀스](../../A_01_user/A_01_50-sequence/SCN_A_01_04_change_user_status.md)
- [Auth 서비스 설계](../A_300_30-service/README.md)
- [Auth 영속성 설계](../A_300_20-persistence/README.md)
