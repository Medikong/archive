---
id: API.A.300-15
title: 로그아웃 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-20
---

# API.A.300-15 로그아웃

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/sessions/logout` |
| operationId | `logoutSession` |
| 역할 | 웹·모바일의 현재 Session과 refresh family를 폐기한다. |
| API 유형 | Command |
| 인증 | 웹 `__Secure-dm_refresh` cookie·CSRF·Origin 또는 모바일 `X-Refresh-Token` header |
| 권한 | 제출한 credential이 가리키는 현재 Session 또는 refresh family만 폐기 가능 |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_15_logout_session.yaml](openapi/paths/API_A_300_15_logout_session.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

웹과 모바일의 조건부 credential은 OpenAPI의 security와 `x-channel-security`를 함께 기준으로 한다. 모바일 credential은 `MobileRefreshToken` security scheme으로 표현하며 request body는 생략하거나 빈 객체만 사용한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [인증 시퀀스 인덱스](../A_300_50-sequence/README.md) |

## 책임과 경계

- 웹은 refresh cookie가 가리키는 Session과 refresh family를 폐기하고 cookie를 만료시킨다.
- 모바일은 `X-Refresh-Token` header로 제출한 refresh credential이 속한 refresh family와 Session을 폐기한다.
- 다른 사용자, 다른 Session 또는 임의 family를 요청값으로 지정할 수 없다.
- 모든 기기의 Session 일괄 폐기는 별도 내부·운영 명령의 책임이다.

## 보안과 개인정보

- 웹은 refresh cookie, CSRF token, Origin과 Fetch Metadata를 함께 검증한다.
- 모바일 refresh token은 `X-Refresh-Token` header credential이며 원문을 로그, trace, event와 IdempotencyRecord에 남기지 않는다.
- 성공 여부로 Session의 과거 상태나 다른 credential 존재를 공개하지 않는다.
- 웹 성공 응답은 refresh cookie를 즉시 만료시키고 body를 반환하지 않는다.

## 처리 규칙

1. 웹 security 조합 또는 모바일 `MobileRefreshToken` security scheme으로 요청 채널과 credential을 검증한다.
2. credential에서 폐기할 Session과 refresh family를 서버 측에서 결정한다.
3. 두 채널 모두 해당 Session과 refresh family 전체를 폐기한다.
4. 감사 OutboxEvent를 저장하고 웹에는 cookie 만료 header를 반환한다.
5. 이미 폐기된 같은 대상을 다시 요청해도 완료로 처리한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `Session`, `SessionCredential`, `ReauthenticationProof`, `IdempotencyRecord` | 우회 | Session 폐기와 관련 credential·proof 무효화를 PostgreSQL 트랜잭션으로 확정해야 한다. |
| `SessionStatusProjection` | 무효화 | 커밋 뒤 key를 삭제하지 않고 `revoked` 상태를 기록해 오래된 access JWT가 cache miss로 되살아나지 않게 한다. |
| refresh token과 reauth proof | 사용하지 않음 | bearer secret과 일회용 proof는 Redis에 저장하지 않는다. |

## 상태 변경과 트랜잭션

- 시작 상태는 active 또는 이미 revoked인 Session·SessionCredential이다.
- Session, active credential 또는 refresh family 폐기와 감사 OutboxEvent를 한 트랜잭션에 저장한다.
- 미소비 ReauthenticationProof도 Session 폐기 범위에 맞춰 무효화한다.
- 성공 종료 상태는 `Session.revoked`와 대상 credential의 `revoked`다.
- Query나 외부 Provider 호출 없이 로컬 transaction으로 완료한다.

## 멱등성과 동시성

- 멱등 범위는 API ID, 해석된 Session 또는 refresh family와 `Idempotency-Key`다.
- 모바일 token 원문 대신 lookup digest와 request HMAC fingerprint만 저장한다.
- 같은 key와 같은 요청은 기존 `204` 완료 결과를 반환한다.
- 같은 key와 다른 요청은 충돌로 거부한다.
- Session row lock과 상태 조건 update로 동시 로그아웃을 한 번의 상태 변경으로 수렴시킨다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ErrorResponse schema와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| credential 누락·오류 | 대상 Session 존재 여부를 숨긴다. | 로컬 credential을 삭제하고 다시 로그인한다. |
| 웹 CSRF·Origin 불일치 | cookie 유효 여부를 추가로 공개하지 않는다. | 새 인증 컨텍스트에서 다시 요청한다. |
| 같은 key의 다른 요청 | 새 폐기 대상을 실행하지 않는다. | 원래 key와 요청을 사용한다. |
| 저장소 장애 | 성공으로 가장하지 않는다. | 로컬 credential 보존 여부를 정책에 따라 결정하고 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `LogoutCurrentSessionHandler` |
| Aggregate / Entity | `Session`, `SessionCredential`, `ReauthenticationProof` |
| Repository / Read Model | Session·Credential Repository, `IdempotencyRecord` |
| Port / Adapter | 해당 없음 |
| Domain / Integration Event | 로그아웃·Session 폐기 감사 OutboxEvent |

## 관측성과 운영

- 로그에는 session ID, client channel, 폐기 범위와 일반화된 결과만 남긴다.
- 로그아웃 요청 수, 폐기 성공·재시도·오류와 transaction latency를 관측한다.
- credential 해석, row lock, 폐기와 outbox 저장을 span으로 기록한다.
- cookie와 `X-Refresh-Token` header는 sampling하지 않으며 빈 요청 body도 수집하지 않는다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 웹은 body 없이 Session, refresh family와 cookie를 폐기한다.
- 모바일은 body credential을 받지 않고 `X-Refresh-Token`으로 제출한 token family만 폐기한다.
- body를 생략하거나 빈 객체로 제출한 요청만 허용한다.
- 이미 폐기된 대상과 동시 요청이 같은 완료 결과로 수렴한다.
- 다른 Session이나 family를 지정할 수 없다.
- credential 원문이 로그, trace, metric label과 event에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [인증 처리 시퀀스 인덱스](../A_300_50-sequence/README.md)
- 관련 API: `API.A.300-14`, `API.A.300-16`
- 전용 로그아웃 시퀀스는 아직 없으며 여러 참여자가 추가되면 `A_300_50-sequence`에서 관리한다.

## 호환성과 변경 정책

- 성공 응답의 비민감 header 추가는 하위 호환으로 처리한다.
- 모바일 `X-Refresh-Token` 전달 위치나 폐기 범위 변경은 새 API 버전으로 제공한다.
- 웹·모바일 endpoint 분리는 기존 경로의 deprecation 기간을 거친다.

## 확인 필요

- 없음.
