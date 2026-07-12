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
updated: 2026-07-10
---

# API.A.300-16 현재 인증 컨텍스트 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/auth/context` |
| operationId | `getAuthContext` |
| 역할 | 현재 요청의 익명 또는 인증 상태와 안전한 Session 요약을 반환한다. |
| API 유형 | Query |
| 인증 | 선택적 웹 Session cookie 또는 모바일 access JWT, 두 credential의 동시 제출은 금지 |
| 권한 | 본인 요청 credential이 가리키는 인증 컨텍스트만 조회 가능 |
| 노출 범위 | public |
| 멱등성 | 해당 없음 |
| 캐시 | `no-store`, `Vary: Cookie, Authorization` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_16_get_auth_context.yaml](openapi/paths/API_A_300_16_get_auth_context.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

익명·웹·모바일 응답의 `oneOf`, 선택적 security, `x-credential-precedence: reject_multiple`, 응답 header, HTTP 상태와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [인증 시퀀스 인덱스](../../../80-sequence/A_300_auth/README.md) |

## 책임과 경계

- credential이 없거나 선택적 credential이 유효하지 않으면 익명 컨텍스트를 반환한다.
- 유효한 credential이면 `user_id`, role, Session 요약과 연결된 인증 수단 종류만 반환한다.
- 웹 응답에는 현재 Session credential에 바인딩된 CSRF token을 포함한다.
- 보호 API의 인증·인가 결과를 대신하거나 구매 가능 여부를 판단하지 않는다.

## 보안과 개인정보

- 이메일, 휴대폰 번호, Identity ID와 credential 원문을 반환하지 않는다.
- 변조·만료·폐기된 선택적 credential도 `authenticated=false`로 일반화한다.
- 웹 Session cookie와 모바일 Bearer token이 함께 오면 어느 쪽도 우선하지 않고 `AUTH_MULTIPLE_CREDENTIALS`로 거부한다.
- 응답은 공유 캐시와 브라우저 저장을 막고 credential 종류에 따라 cache key가 섞이지 않게 한다.
- 이 Query 결과를 다른 보호 API의 권한 근거로 재사용하지 않는다.

## 처리 규칙

1. 웹 Session cookie, 모바일 Bearer token 또는 익명 요청을 식별한다.
2. cookie와 Bearer token이 함께 있으면 credential을 해석하기 전에 요청을 거부한다.
3. 하나의 credential이 있으면 서명, Session 상태, 만료와 UserAuthState를 검증한다.
4. 유효하지 않은 선택적 credential은 오류 대신 익명 컨텍스트로 변환한다.
5. 유효한 Session이면 role, 인증 수단 종류와 채널별 Session 요약을 조합한다.
6. 웹 Session에만 현재 credential에서 도출한 CSRF token을 반환한다.

## 상태 변경과 트랜잭션

- 인증 상태를 조회하는 Query이며 Session, credential, AccessGrant와 IdentityLink를 변경하지 않는다.
- 조회 일관성은 하나의 request snapshot 안에서 Session, UserAuthState와 AccessGrant version을 확인한다.
- `last_seen_at` 갱신이 필요하면 이 Query transaction과 분리된 정책으로 처리한다.
- OutboxEvent와 외부 부수 효과를 만들지 않는다.

## 멱등성과 동시성

- GET Query이므로 `Idempotency-Key`를 받지 않는다.
- 같은 credential과 같은 상태 snapshot은 같은 인증 의미를 반환한다.
- 조회 중 Session이 폐기되면 최종 상태 확인에서 익명 컨텍스트로 수렴한다.
- 동적 식별자는 cache key나 metric label로 사용하지 않는다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ProblemDetails schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| credential 없음·만료·변조·폐기 | 원인을 구분하지 않고 익명 컨텍스트를 반환한다. | 보호 행동에서 로그인 절차를 시작한다. |
| cookie와 Bearer token 동시 제출 | credential 우선순위와 유효성을 공개하지 않는다. | 하나의 credential만 남겨 다시 요청한다. |
| 사용자 제한 또는 비활성 | 세부 제한 사유를 반환하지 않는다. | 새 Session을 발급하지 않고 지원 절차를 안내한다. |
| 필수 저장소 장애 | 인증 상태를 추측하지 않는다. | 선택적 개인화 영역만 대체 상태로 표시한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `ResolvePrincipalHandler`와 인증 컨텍스트 Query 조합 |
| Aggregate / Entity | `Session`, `SessionCredential`, `AccessGrant`, `IdentityLink`, `UserAuthState` |
| Repository / Read Model | Session·AccessGrant·인증 수단 요약 Read Model |
| Port / Adapter | access JWT verifier |
| Domain / Integration Event | 해당 없음 |

## 관측성과 운영

- 로그에는 인증 여부, channel, credential 상태의 일반화 범주만 남긴다.
- 익명·웹·모바일 응답 비율과 선택 credential 검증 실패율을 관측한다.
- credential 검증과 read model 조회를 별도 span으로 기록한다.
- 응답 body, CSRF token, cookie와 Authorization header는 sampling하지 않는다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 익명, 유효 웹, 유효 모바일 응답이 각 `oneOf` schema를 통과한다.
- 만료·변조·폐기 credential이 모두 익명 응답으로 일반화된다.
- cookie와 Bearer token을 함께 제출하면 `400 AUTH_MULTIPLE_CREDENTIALS`를 반환한다.
- 웹 응답에만 CSRF token이 있으며 credential 회전 뒤 값이 바뀐다.
- 응답이 `no-store`와 올바른 `Vary` header를 포함한다.
- 개인정보와 credential이 응답·로그·trace·metric label에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [인증 처리 시퀀스 인덱스](../../../80-sequence/A_300_auth/README.md)
- 관련 API: 보호 행동을 시작하는 모든 API와 선택적 개인화 BFF
- 전용 조회 시퀀스는 없으며 여러 참여자가 추가되면 `80-sequence`에서 관리한다.

## 호환성과 변경 정책

- 새로운 비민감 role·Session metadata 추가는 하위 호환으로 처리한다.
- `authenticated` 의미와 익명 fallback 정책 변경은 새 버전으로 제공한다.
- linked method의 원문 식별자 노출은 호환성 추가가 아니라 보안 정책 변경으로 금지한다.

## 확인 필요

- 제한 사용자에게 익명 응답을 줄지 별도 일반화 상태를 줄지는 UI 지원 정책과 함께 확정한다.
