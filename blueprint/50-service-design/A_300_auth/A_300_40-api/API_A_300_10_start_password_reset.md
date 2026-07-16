---
id: API.A.300-10
title: 비밀번호 재설정 시작 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, password-reset]
source: local
created: 2026-07-10
updated: 2026-07-16
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# API.A.300-10 비밀번호 재설정 시작

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/password-resets` |
| operationId | `startPasswordReset` |
| 역할 | 이메일 또는 휴대폰 식별자로 실제·decoy PasswordReset을 접수한다. |
| API 유형 | Command |
| 인증 | 웹 사전 인증 cookie와 CSRF·Origin 또는 모바일 auth flow token |
| 권한 | 사전 인증 컨텍스트 소유를 확인하며 로그인 Session은 요구하지 않는다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_10_start_password_reset.yaml](openapi/paths/API_A_300_10_start_password_reset.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

이메일·휴대폰 request `oneOf`, 접수 응답, 오류와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [SCN.A.310-01](../A_300_50-sequence/SCN_A_310_01_password_reset.md) |

## 책임과 경계

- 보장하는 업무 결과: email 또는 phone 입력에 대해 계정 존재 여부를 숨긴 PasswordReset 접수 결과를 만든다.
- 요청 안에서 하지 않는 일: Challenge를 발급·발송하거나 비밀번호를 변경하지 않는다.
- 다른 Context 또는 외부 시스템의 책임: 후속 API와 Virtual Adapter가 선택한 수단의 code 발급·검증을 담당한다.

## 보안과 개인정보

- email과 phone request를 `oneOf`로 분리하고 휴대폰은 공통 PhoneNumber로 E.164 정규화한다.
- 미존재 계정도 identity가 없는 decoy reset으로 같은 응답을 만든다.
- `methodOptions`는 정책상 지원 수단이며 실제 연결 수단을 뜻하지 않는다.
- 식별자 원문과 lookup key를 응답·로그·trace·metric·Event에 넣지 않는다.

## 처리 규칙

1. 사전 인증 proof와 active Intent를 확인한다.
2. request 분기에 따라 email 또는 phone을 정규화한다.
3. reset rate limit과 진행 중 작업 정책을 확인한다.
4. 대상이 있으면 verified email Identity와 active PasswordCredential을 찾고, 없으면 decoy reset을 만든다.
5. `childExpiresAt = min(now + PasswordReset TTL, Intent 생성 시각 + auth-flow 최대 수명)`으로 계산하고 PasswordReset을 만든다.
6. AuthenticationIntent와 active owner proof의 만료를 `childExpiresAt`으로 맞춘 뒤 지원 recovery method와 만료 시각만 동일한 형태로 반환한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `PasswordReset`, `Identity`, `IdentityLink`, `PasswordCredential`, `AuthenticationIntent` | 우회 | 실재 대상과 decoy를 같은 응답으로 처리하고 계정 존재 여부가 cache hit 차이로 드러나지 않게 한다. |
| `AuthenticationPolicySnapshotProjection` (`P`) | 사용 | 재설정 TTL, recovery method와 제한 정책은 공통 snapshot이다. |
| PasswordReset 상태 | 사용하지 않음 | 이 API는 외부 polling을 제공하지 않고 다음 단계가 소유 proof와 DB 상태를 직접 확인한다. |

## 상태 변경과 트랜잭션

- 시작 상태: PasswordReset 없음 또는 정책상 재사용 가능한 진행 작업.
- 성공 종료 상태: `requested` 실제 또는 decoy PasswordReset.
- 실패·만료 상태: rate limit·입력 실패 시 만들지 않고 TTL 경과 시 `expired`로 닫는다.
- PasswordReset, parent Intent·proof 만료 연장, IdempotencyRecord와 접수 감사 OutboxEvent를 같은 트랜잭션에 저장한다.
- PasswordReset과 parent Intent·proof는 같은 `childExpiresAt`을 사용하며 auth-flow 최대 수명을 넘지 않는다.
- 이 요청에서는 Challenge와 delivery outbox를 만들지 않는다.

## 멱등성과 동시성

- 멱등 범위는 `API.A.300-10 + auth flow + Idempotency-Key`다.
- 정규화 식별자를 포함한 canonical body의 keyed HMAC fingerprint만 저장한다.
- 같은 key·같은 요청은 같은 실제·decoy PasswordReset을 반환한다.
- 같은 key·다른 identifier 분기나 값은 충돌한다.
- 같은 Identity의 진행 중 reset은 하나만 허용하고 decoy는 별도 안전한 namespace를 사용한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 error code는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| 계정·연결 수단 부재 | 성공 접수와 구분하지 않는다. | 선택 화면에서 지원 수단으로 계속한다. |
| 입력·Intent 부재·만료 | 식별자 존재 여부를 추가로 공개하지 않는다. | 입력을 확인하거나 새 Intent를 만든다. |
| rate limit·저장소 장애 | 성공 형태로 응답하지 않는다. | `Retry-After`에 따라 같은 key 또는 새 작업으로 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `RequestPasswordResetHandler` |
| Aggregate / Entity | `PasswordReset`, `Identity`, `IdentityLink`, `PasswordCredential`, `AuthenticationIntent` |
| Repository / Read Model | `PasswordResetRepository`, `IdentityRepository`, `IdentityLinkRepository`, `IdempotencyRecordRepository` |
| Port / Adapter | `IdentityProtectorPort`, `AuthenticationPolicyProvider`, `RateLimitPort`, `Clock` |
| Domain / Integration Event | 비밀번호 재설정 접수 감사 OutboxEvent |

## 관측성과 운영

- method, actual·decoy를 외부와 분리한 내부 result, rate-limit과 latency를 관측한다.
- 존재·미존재 계정의 응답 시간 분포 차이를 보안 지표로 감시한다.
- 초기 제한은 identifier blind index와 IP 기준 15분 3회다.
- 식별자와 후속 code의 body sampling을 비활성화한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- email·phone `oneOf` 외의 혼합 request와 추가 속성을 거부한다.
- 존재·미존재 계정의 응답 schema와 상태가 같다.
- `methodOptions`가 실제 연결 수단을 노출하지 않는다.
- 같은 key replay가 PasswordReset을 중복 생성하지 않는다.
- PasswordReset TTL이 auth-flow 최대 수명을 넘지 않으며 parent Intent와 active proof가 PasswordReset 만료 전 먼저 끝나지 않는다.
- 시작 성공만으로 Challenge나 메시지가 발송되지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.310-01 비밀번호 재설정](../A_300_50-sequence/SCN_A_310_01_password_reset.md)
- 관련 API: `API.A.300-10~13`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- 새 recovery identifier type은 기존 email·phone 분기의 의미를 유지한 `oneOf` 분기로 추가한다.
- 실제 연결 수단을 드러내는 응답 변경은 허용하지 않는다.
- PasswordReset TTL 변경은 policy version으로 적용하고 기존 작업을 소급 연장하지 않는다.

## 확인 필요

- PasswordReset TTL, decoy 보존 기간과 진행 중 reset 재사용 정책의 초기값은 확정되지 않았다.
