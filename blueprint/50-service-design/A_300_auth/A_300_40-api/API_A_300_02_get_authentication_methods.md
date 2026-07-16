---
id: API.A.300-02
title: 인증 수단 조회 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, method]
source: local
created: 2026-07-10
updated: 2026-07-16
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# API.A.300-02 인증 수단 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/auth/methods` |
| operationId | `getAuthenticationMethods` |
| 역할 | 현재 채널·정책·Intent 목적에서 선택할 수 있는 인증 수단을 조회한다. |
| API 유형 | Query |
| 인증 | Intent에 묶인 웹 사전 인증 cookie 또는 모바일 auth flow token |
| 권한 | 조회 proof가 query의 Intent를 소유해야 한다. |
| 노출 범위 | public |
| 멱등성 | 읽기 전용이므로 별도 key 없음 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_02_get_authentication_methods.yaml](openapi/paths/API_A_300_02_get_authentication_methods.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

요청 parameter, 응답 schema, HTTP 상태, 오류와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | 공통 인증 진입 Query이며 직접 연결된 단일 시퀀스는 없다. |

## 책임과 경계

- 보장하는 업무 결과: 채널과 정책상 노출 가능한 UI 선택지를 반환한다.
- 요청 안에서 하지 않는 일: 특정 이메일·휴대폰의 가입 여부나 연결 상태를 조회하지 않는다.
- 다른 Context 또는 외부 시스템의 책임: 클라이언트가 선택 결과를 이용해 해당 인증 API를 호출한다.

## 보안과 개인정보

- 웹 GET에는 CSRF token을 요구하지 않고 사전 인증 cookie 소유만 확인한다.
- 모바일은 Intent에 묶인 auth flow token을 확인한다.
- 소유 불일치와 Intent 부재는 같은 not-found 공개 오류로 합친다.
- 이메일, 휴대폰, Identity ID와 계정 존재 여부를 응답·로그·trace에 넣지 않는다.

## 처리 규칙

1. query의 Intent ID와 사전 인증 proof binding을 확인한다.
2. Intent가 active이고 만료되지 않았는지 확인한다.
3. 채널, Intent 목적과 활성 AuthenticationPolicy를 적용한다.
4. 계정별 상태가 아닌 지원 인증 수단 목록만 반환한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `AuthenticationIntent` | 우회 | 사용자별 소유 proof, active 상태와 만료 시각을 요청마다 PostgreSQL에서 확인한다. |
| `AuthenticationPolicySnapshotProjection` (`P`) | 사용 | 지원 인증 수단은 활성 정책 snapshot을 여러 요청이 공통으로 읽는다. |

## 상태 변경과 트랜잭션

- Query Handler는 AuthenticationIntent와 정책 read model을 읽기만 한다.
- Intent, Identity, Session과 감사 상태를 변경하지 않는다.
- 외부 Provider나 사용자 Context를 호출하지 않는다.

## 멱등성과 동시성

- 안전한 GET이며 IdempotencyRecord를 만들지 않는다.
- 같은 snapshot에서 같은 Intent와 정책은 같은 결과를 반환한다.
- 정책 변경과 동시에 조회하면 각 요청이 읽은 일관된 policy version을 적용한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 error code는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| Intent 부재 또는 소유 불일치 | 두 조건을 구분하지 않는다. | 새 Intent를 만든다. |
| Intent 만료 | 내부 만료 정책을 노출하지 않는다. | 새 Intent를 만든다. |
| 인증 수단 비활성 | 계정 상태로 해석할 정보를 주지 않는다. | 반환된 활성 수단을 다시 조회한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `GetAuthenticationEntryContextHandler` |
| Aggregate / Entity | `AuthenticationIntent`, `AuthenticationPolicy` |
| Repository / Read Model | `AuthenticationIntentRepository`, policy snapshot |
| Port / Adapter | `AuthenticationPolicyProvider` |
| Domain / Integration Event | 읽기 전용이므로 없음 |

## 관측성과 운영

- 인증 수단 조회 수, Intent 만료와 수단 비활성 결과를 일반화된 label로 집계한다.
- query body는 없으며 응답 body sampling도 비활성화한다.
- trace에는 Intent 원문 대신 request ID와 결과 범주만 기록한다.
- 캐시는 사용하지 않고 DB·정책 snapshot을 기준으로 한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- 웹 GET에 CSRF token 없이 소유 cookie로 조회할 수 있다.
- 다른 proof로 조회하면 Intent 존재 여부를 노출하지 않는다.
- 응답에 계정·Identity 연결 여부와 연락처가 포함되지 않는다.
- 정책이 비활성화한 수단을 활성 선택지로 반환하지 않는다.

## 연관 시퀀스

- 이 Query는 가입·로그인의 인증 수단 선택 전에 사용한다.
- 여러 참여자의 Mermaid 다이어그램은 [인증 시퀀스 인덱스](../A_300_50-sequence/README.md)에서 관리한다.

## 호환성과 변경 정책

- 새 인증 수단 type은 기존 enum 의미를 유지한 채 추가한다.
- 기존 type 제거는 deprecation과 클라이언트 사용량 확인 뒤 진행한다.
- 계정별 연결 상태를 노출하는 변경은 이 Endpoint가 아니라 보호된 별도 API로 설계한다.

## 확인 필요

- MVP 이후 Provider 수단을 노출할 조건과 type 이름은 후속 정책에서 확정한다.
