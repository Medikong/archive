---
id: API.A.300-16
title: 현재 인증 컨텍스트 조회 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-16
---

# API.A.300-16 현재 인증 컨텍스트 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/auth/context` |
| operationId | `getAuthContext` |
| 역할 | Istio가 검증한 Principal과 안전한 Session 요약을 반환한다. |
| API 유형 | Query |
| 인증 | 웹·모바일 공통 Bearer access JWT 필수 |
| 권한 | 본인 JWT가 가리키는 active Session만 조회 가능 |
| 노출 범위 | protected |
| 멱등성 | 해당 없음 |
| HTTP 응답 캐시 | `no-store`, `Vary: Authorization` |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_16_get_auth_context.yaml](openapi/paths/API_A_300_16_get_auth_context.yaml)
- JWT와 Istio 검증 기준: [SD.A.300.JWT](../jwt-jwks-istio.md)

## 책임과 경계

- Istio와 auth-service HTTP ext_authz Adapter가 검증한 `sub`, `sid`, `jti`와 SessionStatusService의 active 판정을 전제로 한다.
- 응답에는 `user_id`, Session 요약과 연결된 인증 수단 종류만 포함한다.
- role, permission, membership, seller 상태, 업무 ACL과 사용자 프로필을 반환하지 않는다.
- 보호 API의 업무 인가나 구매 가능 여부를 대신 판단하지 않는다.

## 처리 규칙

1. Istio가 access JWT 서명, issuer, audience, 만료와 필수 claim을 검증한다.
2. HTTP ext_authz Adapter가 Bearer JWT를 다시 검증한 뒤, SessionStatusService가 `sub`, `sid`, `jti`로 Session active 여부와 `user_id` 일치를 확인한다.
3. Istio가 외부 내부용 헤더를 제거하고 `X-User-Id`, `X-Session-Id`, `X-Token-Id`를 만든다.
4. Auth는 해당 Session과 IdentityLink 요약을 읽어 인증 컨텍스트를 반환한다.

credential이 없거나 JWT·Session이 유효하지 않으면 익명 응답으로 바꾸지 않고 `401`을 반환한다. 웹앱 재실행 시에는 먼저 `API.A.300-14`로 refresh cookie를 회전해 새 access JWT를 받은 뒤 이 API를 호출한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `SessionStatusProjection` | 사용 | 보호 API의 핵심 조회이므로 Redis에서 Session 상태를 먼저 확인하고 cache miss 때 PostgreSQL을 조회한다. Redis와 PostgreSQL에서 모두 상태를 확정하지 못하면 요청을 거부한다. |
| `IdentityLink`, `UserAuthState` 요약 | 우회 | 연결된 인증 수단과 계정 상태는 개인정보·보안 상태가 바뀐 직후에도 정확해야 하므로 PostgreSQL에서 직접 조회한다. |
| 최종 인증 context 응답 | 사용하지 않음 | Session 상태 projection 외에 조합된 응답 전체를 서버에서 별도 cache하지 않아 서로 다른 사용자의 context가 섞일 가능성을 없앤다. |

## 상태 변경과 트랜잭션

- Session, credential, IdentityLink와 UserAuthState를 변경하지 않는다.
- `last_seen_at` 같은 관측성 갱신이 필요하면 Query transaction과 분리한다.
- OutboxEvent와 외부 부수 효과를 만들지 않는다.

## 보안과 개인정보

- 이메일, 휴대폰 번호, Identity ID와 credential 원문을 반환하지 않는다.
- Authorization header와 응답 body를 로그·trace·metric에 기록하지 않는다.
- 이 Query 결과를 다른 요청의 인가 proof로 재사용하지 않는다.
- Seller나 다른 업무 Context는 `user_id`를 기준으로 자신의 인가 원장을 별도로 확인한다.

## 오류와 복구

| 조건 | 결과 | 클라이언트 복구 |
| --- | --- | --- |
| access JWT 없음·변조·만료 | `401 AUTH_SESSION_REQUIRED` | 웹은 refresh를 시도하고 모바일은 저장된 refresh token을 사용한다. |
| Session 만료·폐기·`sub` 불일치 | `401 AUTH_SESSION_REVOKED` | credential을 삭제하고 다시 로그인한다. |
| JWKS 또는 Session 상태 확인 불가 | `503 AUTH_SERVICE_UNAVAILABLE` | 인증 성공으로 간주하지 않고 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Handler | `ResolveAuthenticationPrincipalHandler`, `GetAuthContextHandler` |
| Aggregate / Entity | `Session`, `SessionCredential`, `IdentityLink`, `UserAuthState` |
| Repository / Read Model | Session·인증 수단 요약 Read Model |
| Port / Adapter | `SessionStatusService` |
| Domain / Integration Event | 해당 없음 |

## 검증 항목

- Bearer access JWT 없이 호출하면 `401`이다.
- 웹과 모바일이 같은 응답 schema를 사용하고 channel만 Session metadata로 구분한다.
- 응답과 내부 헤더에 role, permission, membership과 개인정보가 없다.
- 폐기된 Session의 JWT는 남은 TTL과 관계없이 차단된다.
- 응답이 `no-store`와 `Vary: Authorization`을 포함한다.

## 연관 시퀀스

- [SCN.A.300-05 웹 JWT 인증과 Session 회전](../A_300_50-sequence/SCN_A_300_05_web_jwt_authentication.md)
