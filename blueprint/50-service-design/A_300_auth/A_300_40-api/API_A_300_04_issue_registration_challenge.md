---
id: API.A.300-04
title: 가입 Challenge 발급 API
type: service-design-api-endpoint
status: draft
tags: [service-design, auth, api, registration, challenge]
source: local
created: 2026-07-10
updated: 2026-07-16
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
---

# API.A.300-04 가입 Challenge 발급

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/registrations/{registrationId}/challenges` |
| operationId | `issueRegistrationChallenge` |
| 역할 | 가입에 필요한 이메일 또는 휴대폰 소유 확인 Challenge를 발급·재발급한다. |
| API 유형 | Command |
| 인증 | 웹 사전 인증 cookie와 CSRF·Origin 또는 모바일 auth flow token |
| 권한 | 현재 proof가 path의 Registration을 소유해야 한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수. 사용자가 새 발송을 의도하면 새 key 사용 |
| HTTP 응답 캐시 | `no-store` |
| 호환성 | `/api/v1`, deprecation 없음 |

## HTTP 명세 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_04_issue_registration_challenge.yaml](openapi/paths/API_A_300_04_issue_registration_challenge.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

요청·응답, 헤더, HTTP 상태, 오류와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

| 구분 | 식별자와 경로 |
| --- | --- |
| 요구사항 | [REQ.A.05](../../../00-requirements/REQ_A_05_auth_member.md) |
| UC | [UC.A.300](../../../30-uc/UC_A_300_auth_member.md) |
| BC | [BC.A.300](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md) |
| 도메인 | [SD.A.30010](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md) |
| 영속성 | [SD.A.30020](../A_300_20-persistence/README.md) |
| 서비스 | [SD.A.30030](../A_300_30-service/README.md) |
| 시퀀스 | [SCN.A.01-01](../../../80-sequence/A_01_user/SCN_A_01_01_user_provisioning_auth_link.md) |

## 책임과 경계

- 보장하는 업무 결과: method에 맞는 Challenge와 비동기 delivery outbox를 원자적으로 만든다.
- 요청 안에서 하지 않는 일: Provider 응답을 기다리거나 소유 확인 성공으로 처리하지 않는다.
- 다른 Context 또는 외부 시스템의 책임: Outbox relay와 Virtual Adapter가 이메일·SMS 메시지를 전달한다.

## 보안과 개인정보

- Registration 소유·상태를 확인하고 다른 소유자의 리소스 존재 여부를 숨긴다.
- destination 원문과 code를 응답·로그·trace·metric·감사 Event에 넣지 않는다.
- 이메일과 휴대폰 방식 모두 MVP에서는 code 입력으로 검증한다.
- delivery payload는 Challenge TTL 이내의 암호화 reference로만 relay에 전달한다.

## 처리 규칙

1. Registration 소유 proof와 `pending_verification` 상태를 확인한다.
2. 요청 method를 `signup_email` 또는 `signup_phone` purpose로 변환한다.
3. 재발급 간격, 발송 횟수와 대상·IP·device rate limit을 확인한다.
4. 이전 issued Challenge가 있으면 revoked로 닫고 새 Challenge를 만든다.
5. Challenge와 delivery outbox를 같은 트랜잭션에 저장한다.

## 저장 모델과 캐시

저장 구조는 [영속성 설계](../A_300_20-persistence/README.md#저장-모델)와 [Redis projection models](../A_300_20-persistence/README.md#redis-projection-models)를 기준으로 한다.

| 저장 모델 | 전략 | 적용 근거 |
| --- | --- | --- |
| `Registration`, `VerificationChallenge`, `IdempotencyRecord`, `OutboxEvent` | 우회 | 기존 Challenge 폐기와 신규 발급의 동시성을 PostgreSQL row lock으로 보장해야 한다. |
| `AuthenticationPolicySnapshotProjection` (`P`) | 사용 | 재발급 간격, 횟수와 Challenge TTL은 반복 조회되는 정책이다. |
| `RegistrationStatusProjection` (`R`) | 갱신 | 발급된 method 상태와 재시도 가능 시각을 상태 조회 API에 즉시 반영한다. |

## 상태 변경과 트랜잭션

- 시작 상태: 대상 method가 미검증이고 Registration이 `pending_verification`이다.
- 성공 종료 상태: 새 `issued` Challenge와 Registration의 현재 Challenge 참조.
- 실패·만료 상태: Registration 만료 시 새 Challenge를 만들지 않는다.
- 기존 Challenge 폐기, 새 Challenge, Registration 참조, IdempotencyRecord와 delivery OutboxEvent를 함께 저장한다.
- Provider 호출은 relay가 커밋 뒤 수행한다.

## 멱등성과 동시성

- 멱등 범위는 `API.A.300-04 + Registration + method + Idempotency-Key`다.
- 같은 key·같은 method는 같은 Challenge metadata를 반환하고 메시지를 다시 보내지 않는다.
- 같은 key·다른 요청은 충돌한다.
- 새 재발급은 새 key를 사용하되 resend interval과 최대 발송 횟수를 다시 확인한다.
- context와 purpose별 active Challenge는 하나만 허용하고 Registration version으로 경쟁을 제어한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 error code는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| 소유·Registration 부재 | 존재 여부를 더 공개하지 않는다. | 현재 가입 화면에서 다시 시작한다. |
| Registration 만료 | Challenge 만료와 혼동하지 않는다. | 새 Registration을 시작한다. |
| 재발급 제한 또는 장애 | destination과 내부 threshold를 숨긴다. | `Retry-After` 이후 새 key로 요청한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command / Query Handler | `RequestVerificationChallengeHandler` |
| Aggregate / Entity | `Registration`, `VerificationChallenge` |
| Repository / Read Model | `RegistrationRepository`, `VerificationChallengeRepository`, `IdempotencyRecordRepository`, `OutboxEventRepository` |
| Port / Adapter | `SecretGeneratorPort`, `AuthenticationPolicyProvider`, `RateLimitPort`, Email/SMS sender Adapter |
| Domain / Integration Event | `EVT.A.300-22 인증 Challenge 발급됨`, `VerificationDeliveryRequested` |

## 관측성과 운영

- purpose, channel, result, policy version만 로그·trace와 metric label에 사용한다.
- 발급, 재발급, rate-limit, delivery backlog와 provider 결과를 분리해 관측한다.
- 초기 제한은 destination·purpose·IP 기준 15분 3회, 하루 10회다.
- Provider timeout은 worker당 초기 3초이며 사용자 API 트랜잭션과 분리한다.

## 검증 항목

- OpenAPI lint와 bundle이 성공한다.
- Registration이 요구하는 미검증 method에만 발급한다.
- 같은 key 재시도에서 Challenge와 delivery가 중복되지 않는다.
- 새 key 재발급은 이전 Challenge를 revoked로 닫는다.
- Registration 만료는 `AUTH_REGISTRATION_EXPIRED`로 처리한다.
- destination과 code 원문이 관측 데이터와 Event에 남지 않는다.

## 연관 시퀀스

- 시퀀스 문서: [SCN.A.01-01 회원가입과 자동 로그인](../../../80-sequence/A_01_user/SCN_A_01_01_user_provisioning_auth_link.md)
- 관련 API: `API.A.300-03~06`, `API.A.300-28`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- 새 가입 검증 method는 기존 enum 의미를 유지한 별도 값으로 추가한다.
- code가 아닌 link proof 도입은 검증 request schema와 클라이언트 입력 형식 변경으로 별도 버전에서 처리한다.
- resend 정책 변경은 policy version으로 운영하며 wire schema를 바꾸지 않는다.

## 확인 필요

- 이메일·SMS Challenge TTL과 resend interval의 최종 운영값은 확정되지 않았다.
