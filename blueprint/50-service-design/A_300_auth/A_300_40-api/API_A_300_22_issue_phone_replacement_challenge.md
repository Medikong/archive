---
id: API.A.300-22
title: 휴대폰 번호 교체 Challenge 발급 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-10
---

# API.A.300-22 휴대폰 번호 교체 Challenge 발급

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/phone-replacements/{replacementId}/challenges` |
| operationId | `issuePhoneReplacementChallenge` |
| 역할 | 교체 대상으로 고정된 새 휴대폰 번호에 SMS Challenge를 발급한다. |
| API 유형 | Command |
| 인증 | requested Link 소유 사용자 Session, 웹 CSRF·Origin 또는 모바일 Bearer |
| 권한 | 현재 사용자·Session이 requested Link의 소유 binding과 같아야 한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, 미지원 중단 상태 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_22_issue_phone_replacement_challenge.yaml](openapi/paths/API_A_300_22_issue_phone_replacement_challenge.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

요청·응답 schema, 헤더, HTTP 상태, 오류와 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [SD.A.30010 인증 도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- [SD.A.30020 인증 영속성 설계](../A_300_20-persistence/README.md)
- [SD.A.30030 인증 서비스 설계](../A_300_30-service/README.md)
- [SCN.A.300-03 휴대폰 번호 교체](../../../80-sequence/A_300_auth/SCN_A_300_03_phone_replacement.md)

## 책임과 경계

- requested Link에 저장된 새 번호만 발송 대상으로 사용한다.
- Challenge, delivery payload와 발송 OutboxEvent 생성을 보장한다.
- 번호 소유 확인과 기존·신규 Link 전환은 이 Endpoint에서 수행하지 않는다.
- Virtual SMS Adapter의 실제 전달은 비동기 relay 책임이다.

## 보안과 개인정보

- path ID만으로 접근하지 않고 현재 Session과 requested Link binding을 검증한다.
- 응답은 마스킹 수신지만 포함하며 휴대폰 번호 원문을 반환하지 않는다.
- 인증번호, delivery payload, 휴대폰 원문을 로그·trace·metric·event에 넣지 않는다.
- 존재하지 않거나 다른 사용자 소유인 requested Link는 같은 공개 오류로 처리한다.

## 처리 규칙

1. 요청자 Session과 UserAuthState를 확인한다.
2. requested Link의 소유자, 상태와 완료 기한을 검증한다.
3. `phone_change` 목적과 `sms_code` channel의 정책을 고정한다.
4. Challenge와 암호화 delivery payload를 만들고 비동기 발송을 요청한다.
5. 마스킹 수신지와 만료·재발송 가능 시각을 반환한다.

## 상태 변경과 트랜잭션

- Challenge, delivery payload, IdempotencyRecord와 발송 OutboxEvent를 같은 트랜잭션에 저장한다.
- `context_id`는 `replacementId`와 같고 requested Link의 새 Identity를 참조한다.
- provider 호출은 커밋 뒤 relay가 수행하며 provider 실패가 Challenge 중복 생성으로 이어지지 않는다.
- 만료된 requested Link에는 새 Challenge를 만들지 않는다.

## 멱등성과 동시성

- 멱등 범위는 API ID, requested Link, 현재 사용자, `Idempotency-Key`다.
- 같은 key와 같은 요청은 같은 Challenge와 발송 결과 참조를 반환하고 SMS를 다시 보내지 않는다.
- 새 인증번호 발송을 명시적으로 원하면 재발송 제한 이후 새 key를 사용한다.
- Link row version과 목적별 발급 제한을 함께 확인해 동시 발급을 제한한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 ProblemDetails 계약은 OpenAPI를 기준으로 한다.

- requested Link가 없거나 소유 binding이 다르면 존재 여부를 숨긴다.
- requested Link 완료 기한이 지나면 새 교체 작업부터 다시 시작한다.
- 발급 제한에 걸리면 공개된 재시도 시각 이후 새 key로 요청한다.
- 비동기 발송 실패는 같은 delivery ID로 정책 범위에서 재시도한다.

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command Handler | `RequestVerificationChallengeHandler` |
| Aggregate / Entity | `IdentityLink`, `VerificationChallenge`, `Session` |
| Repository | `IdentityLinkRepository`, `VerificationChallengeRepository`, `IdempotencyRepository` |
| Port / Adapter | `VerificationDeliveryPort`, `VirtualVerificationAdapter` |
| Event | `EVT.A.300-22 인증 Challenge 발급됨`, delivery OutboxEvent |

## 관측성과 운영

- purpose, channel, 결과와 policy version만 관측 데이터에 기록한다.
- `auth_verification_challenge_issued_total{purpose,channel,result}`와 relay 지연을 측정한다.
- destination, code, replacement ID는 로그·trace attribute·metric label에서 제외한다.
- rate limit 응답의 재시도 정보는 OpenAPI 계약을 따른다.

## 검증 항목

- 다른 사용자의 replacement ID로 발급할 수 없다.
- requested Link에 저장되지 않은 번호로 발송할 수 없다.
- 같은 key 재시도가 메시지를 다시 발송하지 않는다.
- 만료된 Link와 발급 제한 초과 요청을 구분해 복구 가능하게 처리한다.
- DB rollback 뒤 Challenge나 OutboxEvent가 부분 저장되지 않는다.

## 연관 시퀀스

- [SCN.A.300-03 휴대폰 번호 교체](../../../80-sequence/A_300_auth/SCN_A_300_03_phone_replacement.md)
- 선행 API: `API.A.300-21`
- 후속 API: `API.A.300-23`, 개발·테스트에서는 `API.A.300-30`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- Challenge 응답에 선택적 표시 필드를 추가하는 변경은 하위 호환이다.
- purpose, channel 또는 재발송 의미 변경은 새 버전에서 처리한다.
- 외부 SMS Provider 도입 여부는 이 HTTP 계약을 바꾸지 않는다.

## 확인 필요

- 휴대폰 번호 변경 Challenge의 TTL, 최대 발급 횟수와 재발송 간격을 확정한다.
- requested Link 만료 오류의 사용자 안내 문구를 확정한다.
