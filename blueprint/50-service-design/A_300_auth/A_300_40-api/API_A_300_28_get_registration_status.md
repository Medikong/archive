---
id: API.A.300-28
title: 회원가입 상태 조회 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-10
---

# API.A.300-28 회원가입 상태 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/auth/registrations/{registrationId}` |
| operationId | `getRegistrationStatus` |
| 역할 | 중단·재시도 뒤 회원가입 단계와 자동 로그인 재개 가능 여부를 조회한다. |
| API 유형 | Query |
| 인증 | 해당 Registration의 웹 사전 인증 cookie, 모바일 flow token 또는 상태 조회 전용 token |
| 권한 | Registration 소유 binding 일치 |
| 노출 범위 | public |
| 멱등성 | 해당 없음 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, 미지원 중단 상태 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_28_get_registration_status.yaml](openapi/paths/API_A_300_28_get_registration_status.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

상태 조회 전용 token 보안 조합과 진행·완료·실패·만료 응답 schema, wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [SD.A.30010 인증 도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- [SD.A.30020 인증 영속성 설계](../A_300_20-persistence/README.md)
- [SD.A.30030 인증 서비스 설계](../A_300_30-service/README.md)
- [SCN.A.300-01 이메일 회원가입과 자동 로그인](../../../80-sequence/A_300_auth/SCN_A_300_01_email_registration.md)

## 책임과 경계

- 소유 확인된 Registration의 현재 단계와 검증 완료 수단을 조회한다.
- terminal 상태도 정상 상태 조회 결과로 반환해 클라이언트가 한 계약으로 처리하게 한다.
- API.A.300-03이 발급한 상태 조회 전용 token으로 사전 인증 credential 정리 뒤에도 짧은 보존 기간 동안 terminal 상태를 확인할 수 있게 한다.
- Session, cookie, access token이나 refresh token을 발급하지 않는다.
- User 생성 상세와 내부 실패 사유를 공개하지 않는다.

## 보안과 개인정보

- 웹 사전 인증 cookie, 모바일 flow token 또는 상태 조회 전용 token과 Registration binding을 검증한다.
- 상태 조회 token은 hash만 저장하고 `registration_status_read` 용도로만 사용하며 회원가입 완료나 Session 발급에는 사용할 수 없다.
- 소유 증명이 없거나 다르면 Registration 부재와 같은 공개 결과를 사용한다.
- 진행 중에는 `user_id`, Identity 원문과 내부 실패 상세를 반환하지 않는다.
- 상태 응답과 소유 proof를 로그·trace·metric label에 넣지 않는다.

## 처리 규칙

1. Registration ID와 허용된 소유 proof 하나를 검증한다.
2. Registration과 필수 Challenge의 현재 상태를 읽는다.
3. 공개 가능한 검증 수단, 재시도 가능 여부와 만료 시각을 계산한다.
4. 진행 상태와 terminal 상태를 모두 같은 성공 envelope로 반환한다.
5. `verified`이면 프론트엔드가 User 생성 API를 호출할 수 있음을 표시한다.

## 상태 변경과 트랜잭션

- 상태를 바꾸지 않는 Query다.
- 조회 중 Registration을 다음 단계로 진행하거나 Session을 발급하지 않는다.
- Challenge 검증과 완료 Command만 Registration 상태를 변경한다.
- 만료 worker의 결과도 terminal 상태로 읽을 뿐 조회가 만료를 직접 수행하지 않는다.
- 상태 조회 token hash와 만료 시각은 Registration에 귀속되며 terminal 전환 뒤 짧은 조회 보존 기간까지만 유지한다.

## 멱등성과 동시성

- GET Query이므로 Idempotency-Key를 받지 않는다.
- 같은 시점의 같은 상태는 같은 공개 필드로 표현한다.
- 상태가 바뀌어도 조회 자체가 중복 부수 효과를 만들지 않는다.
- 사전 인증 credential이 정리된 뒤에는 같은 Registration의 유효한 상태 조회 token만 terminal 조회를 이어 갈 수 있다.
- 가입 완료는 이 Query가 아니라 프론트엔드의 User 생성과 Auth 완료 요청으로 실행한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 ProblemDetails 계약은 OpenAPI를 기준으로 한다.

- Registration 부재, 소유 proof 오류·만료와 binding 불일치는 같은 not-found 오류로 처리한다.
- failed와 expired는 오류 응답으로 바꾸지 않고 재시도 불가 terminal 상태로 반환한다.
- 저장소 장애를 존재하지 않는 Registration이나 빈 상태로 가장하지 않는다.

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Handler | `GetRegistrationStatusHandler` |
| Aggregate / Entity | `Registration`, `VerificationChallenge`, 상태 조회 token hash |
| Read Model | `RM.A.300-05 회원가입 상태` |
| Repository | `RegistrationRepository` |
| Event | Query 자체는 Domain Event를 만들지 않는다. |

## 관측성과 운영

- 공개 상태, 결과와 처리 지연만 집계한다.
- `auth_registration_status_query_total{status,result}`를 관측한다.
- Registration ID, proof, `user_id`와 Identity 참조는 metric label에 넣지 않는다.
- 조회 빈도는 client와 Registration 단위 rate limit으로 제어한다.

## 검증 항목

- 다른 사전 인증 컨텍스트로 상태를 조회할 수 없다.
- 상태 조회 token으로 회원가입 완료 API나 Session 발급을 호출할 수 없다.
- 사전 인증 credential 정리 뒤에도 유효한 상태 조회 token으로 terminal 상태를 읽고, 보존 기간이 끝나면 읽지 못한다.
- Query가 Registration 상태나 Session artifact를 만들지 않는다.
- 모든 진행·terminal 상태가 같은 성공 schema를 통과한다.
- `verified` 응답 뒤 프론트엔드가 User 생성과 Auth 완료를 차례로 실행한다.
- 내부 실패 사유와 `user_id`가 진행 중 응답에 노출되지 않는다.

## 연관 시퀀스

- [SCN.A.300-01 이메일 회원가입과 자동 로그인](../../../80-sequence/A_300_auth/SCN_A_300_01_email_registration.md)
- 연관 API: `API.A.300-03~06`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- 상태별 선택 안내 필드 추가는 하위 호환이다.
- 기존 상태 이름과 terminal 의미 변경은 새 API 버전에서 처리한다.
- 상태 조회 token의 권한을 완료 명령으로 넓히는 변경은 별도 보안 검토와 새 credential 계약이 필요하다.
- 새 내부 단계를 추가할 때 외부 상태로 그대로 노출하지 않고 기존 공개 상태에 매핑한다.

## 확인 필요

- polling rate limit과 권장 재조회 간격을 확정한다.
- failed·expired 상태에서 사용자에게 보여 줄 일반 안내 문구를 확정한다.
