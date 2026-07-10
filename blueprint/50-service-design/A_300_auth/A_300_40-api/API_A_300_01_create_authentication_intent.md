---
id: API.A.300-01
title: AuthenticationIntent 생성 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, intent]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# API.A.300-01 AuthenticationIntent 생성

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/intents` |
| operationId | `createAuthenticationIntent` |
| 역할 | 로그인 전 복귀 위치와 허용된 사용자 행동을 단기 Intent로 보존한다. |
| API 유형 | Command |
| 인증 | 비로그인 bootstrap. 웹은 Origin·Fetch Metadata를, 모바일은 채널·설치 ID·활성화된 attestation 정책을 검증한다. |
| 권한 | 사용자 권한은 없으며 Gateway의 채널별 요청 검증을 통과해야 한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_01_create_authentication_intent.yaml](openapi/paths/API_A_300_01_create_authentication_intent.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

Method/Path, parameter, request/response schema, required 여부, 응답 header, HTTP 상태, 오류와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | 공통 선행 Endpoint이며 직접 연결된 단일 시퀀스는 없다. |

## 책임과 경계

- 보장하는 업무 결과: 검증된 내부 복귀 위치와 `navigation` 또는 `purchase` 행동을 소유 proof에 묶어 보존한다.
- 요청 안에서 하지 않는 일: 로그인, Session 발급, 구매 실행을 하지 않는다.
- 다른 Context 또는 외부 시스템의 책임: BFF가 행동별 최소 입력을 만들고 Gateway가 채널·Origin·attestation을 검증한다.

## 보안과 개인정보

- 외부 URL, scheme-relative URL, 제어 문자와 이중 인코딩 우회 값을 거부한다.
- `purchase`의 허용 필드만 암호화하고 개인정보, 결제 비밀정보와 임의 JSON을 저장하지 않는다.
- owner proof, CSRF token과 모바일 auth flow token 원문을 저장하거나 로그·trace·metric에 남기지 않는다.
- 동일 요청 재생에서도 credential 원문이나 암호화된 응답을 보관하지 않고 새 owner proof와 웹 CSRF token을 발급한다.
- 아직 사전 인증 credential이 없으므로 채널별 조건부 검증은 OpenAPI 설명과 Gateway 정책을 함께 기준으로 한다.

## 처리 규칙

1. `X-Client-Channel`을 검증하고 도메인 채널 `web` 또는 `mobile`로 고정한다.
2. `returnPath`를 정규화하고 내부 allowlist를 적용한다.
3. `intentType`별 schema로 action context를 검증하고 최소 payload만 암호화한다.
4. 같은 key·같은 요청이 완료되어 있으면 기존 Intent를 잠그고 owner proof를 교체하며 웹 CSRF token도 함께 교체한다.
5. 새 요청이면 AuthenticationIntent와 payload reference를 같은 로컬 트랜잭션에 저장한다.
6. 웹에는 사전 인증 cookie와 CSRF token을, 모바일에는 auth flow token을 전달한다.

## 상태 변경과 트랜잭션

- 시작 상태: Intent 없음.
- 성공 종료 상태: `active` AuthenticationIntent.
- 실패·만료 상태: 검증 실패 시 생성하지 않으며 TTL 경과 시 `expired`로 닫는다.
- 같은 로컬 트랜잭션에 AuthenticationIntent, 암호화 ActionIntentPayload, IdempotencyRecord와 OutboxEvent를 저장한다.
- 동일 요청 재생은 기존 Intent와 IdempotencyRecord를 잠근 뒤 proof hash 교체와 이전 proof 무효화를 한 트랜잭션에서 처리한다.
- 외부 채널 검증은 저장 트랜잭션 전에 끝내며 구매 Context를 호출하지 않는다.

## 멱등성과 동시성

- 멱등 범위는 `API.A.300-01 + Idempotency-Key`의 bootstrap namespace다.
- 전체 canonical body의 keyed HMAC fingerprint만 저장한다.
- 같은 key·같은 요청은 같은 Intent를 유지하되 owner proof와 웹 CSRF token을 원자적으로 다시 발급하고 이전 proof를 즉시 무효화한다.
- 멱등 저장소에는 fingerprint와 credential digest만 저장하며 credential 평문, 복호화 가능한 값과 암호화된 응답 replay payload도 저장하지 않는다.
- 같은 key·다른 요청은 새 Intent를 만들지 않고 충돌로 처리한다.
- Intent별 action payload는 최대 하나이며 owner proof hash는 유일해야 한다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code, ProblemDetails와 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| 복귀 경로 또는 행동 schema 오류 | 내부 allowlist와 정규화 사유를 노출하지 않는다. | 안전한 내부 경로와 허용 행동으로 다시 요청한다. |
| 멱등 key 충돌 | 기존 Intent를 덮어쓰지 않는다. | 새 사용자 행동이면 새 key를 사용한다. |
| rate limit 또는 저장소 장애 | 성공 형태를 반환하지 않는다. | 공개된 재시도 시간에 맞춰 다시 요청한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `PreserveAuthenticationIntentHandler` |
| Aggregate / Entity | `AuthenticationIntent`, `ActionIntentPayload` |
| Repository / Read Model | `AuthenticationIntentRepository`, `ActionIntentPayloadRepository`, `IdempotencyRecordRepository` |
| Port / Adapter | `AuthenticationPolicyProvider`, `IdentityProtectorPort`, `RateLimitPort` |
| Domain / Integration Event | `EVT.A.300-02 로그인 의도 보존됨` |

## 관측성과 운영

- 로그·trace에는 action 이름, 결과, request ID만 남기고 경로 원문과 payload는 남기지 않는다.
- Intent 생성 수, 거부 사유 범주, rate-limit과 만료 수를 집계한다.
- 초기 제한은 IP와 bootstrap scope 기준 분당 30회이며 `Retry-After`를 공개한다.
- 인증 Endpoint의 request/response body sampling은 비활성화한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 외부 URL, scheme-relative URL, 제어 문자와 이중 인코딩 우회를 거부한다.
- navigation 요청에는 action payload가 없고 purchase에는 허용 필드만 존재한다.
- 웹과 모바일 응답이 서로의 credential을 포함하지 않는다.
- 같은 key replay가 새 Intent를 만들지 않고 다른 body는 충돌한다.
- 같은 key replay가 새 proof를 전달하고 이전 proof로 이어지는 요청은 거부한다.
- 멱등 저장소에 owner proof·CSRF token 원문이나 암호화된 replay 응답이 남지 않는다.
- 민감 payload와 proof 원문이 관측 데이터에 남지 않는다.

## 연관 시퀀스

- 이 Endpoint는 가입, 이메일·휴대폰 로그인과 비밀번호 재설정 시퀀스의 공통 선행 단계다.
- 여러 참여자의 Mermaid 다이어그램은 [인증 시퀀스 인덱스](../../../80-sequence/A_300_auth/README.md)에서 관리한다.

## 호환성과 변경 정책

- 새로운 `intentType`은 기존 분기의 의미를 바꾸지 않는 별도 `oneOf` schema로 추가한다.
- 기존 action 필드의 삭제·의미 변경과 외부 경로 허용은 새 API 버전으로 처리한다.
- proof 전달 방식 변경은 웹·모바일 클라이언트 이행 기간과 함께 공지한다.

## 확인 필요

- Intent TTL과 purchase action ID의 구체적인 길이·문자 집합은 정책 문서에서 확정해야 한다.
