---
id: API.A.300-23
title: 휴대폰 번호 교체 완료 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-10
---

# API.A.300-23 휴대폰 번호 교체 완료

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/phone-replacements/{replacementId}/complete` |
| operationId | `completePhoneReplacement` |
| 역할 | 새 번호 소유를 확인하고 기존 휴대폰 Link를 닫은 뒤 같은 `user_id`에 새 Link를 활성화한다. |
| API 유형 | Command |
| 인증 | 사용자 Session과 recent auth, 웹 CSRF·Origin 또는 모바일 Bearer |
| 권한 | 현재 사용자·Session이 requested Link와 Challenge binding에 일치해야 한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, 미지원 중단 상태 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_23_complete_phone_replacement.yaml](openapi/paths/API_A_300_23_complete_phone_replacement.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

채널별 credential 회전 응답의 `oneOf`, 반복 `Set-Cookie`, 오류와 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [SD.A.30010 인증 도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- [SD.A.30020 인증 영속성 설계](../A_300_20-persistence/README.md)
- [SD.A.30030 인증 서비스 설계](../A_300_30-service/README.md)
- [SCN.A.300-03 휴대폰 번호 교체](../../../80-sequence/A_300_auth/SCN_A_300_03_phone_replacement.md)

## 책임과 경계

- Challenge를 소비하고 old/new IdentityLink를 원자적으로 전환한다.
- 기존 휴대폰 Link를 인증 근거로 사용한 다른 Session을 정책 범위에서 폐기한다.
- 현재 Session credential은 채널에 맞게 회전해 이 응답에서만 전달한다.
- 사용자 계정 병합, `user_id` 변경과 업무 Context 명령 실행은 제공하지 않는다.

## 보안과 개인정보

- code proof, Session, Challenge와 requested Link 소유 binding을 모두 검증한다.
- 웹은 회전된 HttpOnly cookie와 CSRF token을, 모바일은 새 access/refresh token을 전달한다.
- 회전 전 credential hash는 `rotated_pending_delivery` 복구 판정에만 사용할 수 있고 일반 인증에는 사용할 수 없다.
- code, token, cookie, 전화번호 원문과 암호화 값을 로그·trace·event에 남기지 않는다.
- 다른 사용자에 연결된 새 번호는 기존 계정을 공개하거나 병합하지 않고 거부한다.

## 처리 규칙

1. Session의 유효성, recent auth와 UserAuthState를 확인한다.
2. requested Link와 Challenge의 상태·목적·소유·만료를 검증한다.
3. code proof를 확인하고 Challenge를 소비한다.
4. 기존 Link를 replaced로 닫고 새 Identity와 Link를 같은 `user_id`에 활성화한다.
5. 영향받는 Session을 폐기하고 현재 Session credential을 회전한다.
6. 저장된 client channel에 따라 웹 또는 모바일 결과를 반환한다.

## 상태 변경과 트랜잭션

- Challenge 소비, old Link 종료, new Link 활성화, Session 선별 폐기, 현재 credential 회전, `rotated_pending_delivery`, IdempotencyRecord와 OutboxEvent를 같은 트랜잭션에 저장한다.
- 새 Identity owner와 active Link의 `user_id`는 현재 사용자로 고정한다.
- 일부 Link나 credential만 바뀐 상태로 커밋하지 않는다.
- credential 전달을 확인하면 복구 상태를 완료로 닫고 재전달 자료는 짧은 보존 기간 뒤 폐기한다.
- 감사 broker 전송은 커밋 뒤 relay가 수행한다.

## 멱등성과 동시성

- 멱등 범위는 API ID, requested Link, 현재 사용자·Session, `Idempotency-Key`다.
- 같은 key와 같은 요청은 동일한 논리 교체 결과를 반환하며 새 Link나 Session 회전을 만들지 않는다.
- 같은 key와 다른 Challenge/proof는 충돌로 거부한다.
- Link와 Challenge row lock, Identity active Link 유일성 제약으로 동시 완료를 직렬화한다.
- credential 회전 뒤 응답이 유실된 경우에만 같은 Session, operation, `Idempotency-Key`와 요청 fingerprint를 모두 대조한다.
- 위 binding이 모두 맞고 상태가 `rotated_pending_delivery`일 때만 회전 전 credential hash로 전달 복구를 허용한다. 이 hash를 일반 인증이나 다른 API 재시도에는 절대 허용하지 않는다.

## 예외와 복구 규칙

정확한 HTTP 상태와 ProblemDetails 계약은 OpenAPI를 기준으로 한다.

- Challenge 검증 실패는 내부 남은 횟수와 Identity 존재 여부를 공개하지 않는다.
- requested Link 또는 Challenge가 만료되면 해당 단계부터 다시 시작한다.
- 새 번호가 다른 사용자에게 귀속됐으면 계정 병합 없이 수동 지원을 안내한다.
- 트랜잭션 실패 시 같은 key로 재요청하며 부분 변경을 복구 대상으로 남기지 않는다.
- credential 전달 복구 binding이 하나라도 다르면 회전 전 credential을 받아들이지 않고 다시 인증하도록 한다.
- `rotated_pending_delivery` 복구 TTL이 끝나면 `410 AUTH_SESSION_DELIVERY_EXPIRED`를 반환하고 새 재인증과 교체 요청부터 시작한다.

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command Handler | `VerifyChallengeHandler`, `CompletePhoneChangeHandler` |
| Aggregate / Entity | `Identity`, `IdentityLink`, `VerificationChallenge`, `Session`, `SessionCredential` |
| Repository | `IdentityRepository`, `IdentityLinkRepository`, `VerificationChallengeRepository`, `SessionRepository`, `IdempotencyRepository` |
| Policy | `SessionRevocationPolicy` |
| Event | `EVT.A.300-19`, `EVT.A.300-20`, `EVT.A.300-26`, 감사 OutboxEvent |

## 관측성과 운영

- 교체 결과, 폐기 Session 수, credential channel과 policy version을 관측한다.
- `auth_phone_replacement_completed_total{result,channel}`과 트랜잭션 지연을 측정한다.
- code, token, cookie, 휴대폰 번호와 Identity ID는 관측 데이터에 넣지 않는다.
- 보안 감사에는 변경 경로, 대상 `user_id`, old/new Link 참조와 재인증 결과만 남긴다.

## 검증 항목

- Challenge·Link·Session binding이 하나라도 다르면 교체하지 않는다.
- old Link 종료와 new Link 활성화의 원자성을 검증한다.
- 기존 번호 기반 다른 Session은 폐기하고 현재 Session은 정해진 인증 근거를 유지한다.
- 웹과 모바일 응답이 서로 다른 credential을 섞지 않는다.
- 같은 key 재시도와 동시 요청에서 Link와 credential이 중복되지 않는다.
- `rotated_pending_delivery` 복구는 네 binding이 모두 같은 경우에만 성공하고 회전 전 credential로 일반 API를 호출하면 실패한다.
- 복구 TTL이 끝난 같은-key 요청은 credential을 다시 만들지 않고 `AUTH_SESSION_DELIVERY_EXPIRED`로 종료한다.

## 연관 시퀀스

- [SCN.A.300-03 휴대폰 번호 교체](../../../80-sequence/A_300_auth/SCN_A_300_03_phone_replacement.md)
- 선행 API: `API.A.300-17`, `API.A.300-21`, `API.A.300-22`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- 채널별 응답에 선택 필드를 추가하는 변경은 하위 호환이다.
- credential 전달 방식, Link 종료 의미와 Session 폐기 범위 변경은 새 버전 또는 명시적 정책 version으로 처리한다.
- 외부 SMS Provider 도입은 Challenge 전달 Adapter만 바꾸며 완료 계약을 바꾸지 않는다.

## 확인 필요

- credential 재전달 허용 기간의 초기 운영값을 확정한다.
- 현재 Session을 다시 회전할 때 웹·모바일별 정확한 만료 정책을 확정한다.
