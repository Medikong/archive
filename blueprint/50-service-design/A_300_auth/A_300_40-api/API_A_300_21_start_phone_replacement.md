---
id: API.A.300-21
title: 휴대폰 번호 교체 시작 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-10
---

# API.A.300-21 휴대폰 번호 교체 시작

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /api/v1/auth/phone-replacements` |
| operationId | `startPhoneReplacement` |
| 역할 | 이메일 재인증을 마친 사용자의 새 휴대폰 Identity와 교체용 requested Link를 만든다. |
| API 유형 | Command |
| 인증 | 사용자 Session, 웹 CSRF·Origin 또는 모바일 Bearer, `replace_phone` 목적 ReauthenticationProof |
| 권한 | 현재 Session의 사용자와 proof의 사용자·Session이 같아야 한다. |
| 노출 범위 | public |
| 멱등성 | `Idempotency-Key` 필수 |
| 캐시 | `no-store` |
| 호환성 | `/api/v1`, 미지원 중단 상태 |

## HTTP 계약 원장

- 완전한 OpenAPI 문서: [openapi/openapi.yaml](openapi/openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_21_start_phone_replacement.yaml](openapi/paths/API_A_300_21_start_phone_replacement.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

Method/Path, parameter, request/response schema, required 여부, 헤더, HTTP 상태, 오류와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [SD.A.30010 인증 도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- [SD.A.30020 인증 영속성 설계](../A_300_20-persistence/README.md)
- [SD.A.30030 인증 서비스 설계](../A_300_30-service/README.md)
- [SCN.A.300-03 휴대폰 번호 교체](../../../80-sequence/A_300_auth/SCN_A_300_03_phone_replacement.md)

## 책임과 경계

- 현재 `user_id`의 active 휴대폰 Link를 확인하고 새 번호용 Identity를 예약한다.
- 목적·사용자·Session·만료가 일치하는 proof 소비와 requested Link 생성을 보장한다.
- 이 단계에서는 SMS Challenge를 발급하거나 기존 휴대폰 Link를 닫지 않는다.
- 계정 병합과 `user_id` 변경은 제공하지 않는다.

## 보안과 개인정보

- 웹은 Session cookie, CSRF token, Origin을 함께 검증하고 모바일은 access JWT를 검증한다.
- ReauthenticationProof는 `replace_phone` 목적에 한정하며 한 번만 소비한다.
- 휴대폰 번호 원문, 정규화 값, proof를 로그·trace·metric·event에 남기지 않는다.
- 응답에는 휴대폰 번호와 Identity 원문을 반환하지 않는다.

## 처리 규칙

1. Session과 UserAuthState가 active인지 확인한다.
2. 입력 번호를 정규화하고 영구 귀속·active Link 충돌을 확인한다.
3. ReauthenticationProof의 사용자, Session, 목적, 만료를 검증한다.
4. proof를 소비하고 새 Identity와 `phone_change` requested IdentityLink를 만든다.
5. requested Link ID와 완료 기한을 반환한다.

## 상태 변경과 트랜잭션

- proof 소비, 새 Identity 예약, requested Link, IdempotencyRecord와 감사 OutboxEvent를 같은 트랜잭션에 저장한다.
- requested Link는 기존 active 휴대폰 Link를 `previous_identity_link_id`로 가리킨다.
- 충돌이나 저장 실패 시 proof와 Link 어느 쪽도 부분 반영하지 않는다.
- SMS 발송은 후속 Challenge 발급 Endpoint의 책임이다.

## 멱등성과 동시성

- 멱등 범위는 API ID, 현재 사용자·Session, `Idempotency-Key`다.
- 번호와 proof를 포함한 canonical 요청에는 전용 서버 키 HMAC fingerprint를 적용한다.
- 같은 key와 같은 요청은 기존 `replacementId`와 완료 기한을 반환한다.
- 같은 key에 다른 요청이 오면 새 Identity나 Link를 만들지 않는다.
- active phone Link 조회와 requested Link 생성은 row version과 유일성 제약으로 직렬화한다.

## 예외와 복구 규칙

정확한 HTTP 상태와 ProblemDetails 계약은 OpenAPI를 기준으로 한다.

- proof가 만료·소비됐거나 목적이 다르면 이메일 재인증부터 다시 시작한다.
- 새 번호가 다른 사용자에게 영구 귀속됐거나 active 연결이면 계정 병합 없이 수동 지원을 안내한다.
- 필수 저장소 장애는 성공으로 처리하지 않으며 같은 key 재시도를 허용한다.

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Command Handler | `StartIdentityLinkHandler`의 phone replacement 경로 |
| Aggregate / Entity | `Identity`, `IdentityLink`, `ReauthenticationProof`, `Session` |
| Repository | `IdentityRepository`, `IdentityLinkRepository`, `ReauthenticationProofRepository`, `IdempotencyRepository` |
| Event | `EVT.A.300-18 휴대폰 번호 변경 요청됨`, 감사 OutboxEvent |

## 관측성과 운영

- 결과, 충돌 유형, policy version만 구조화 로그와 trace에 기록한다.
- `auth_phone_replacement_started_total{result}`와 처리 지연을 관측한다.
- 사용자·Session·proof·휴대폰 원문은 metric label과 body sampling에서 제외한다.
- 사용자 단위 rate limit과 운영 경보 기준은 API 공통 정책을 따른다.

## 검증 항목

- 웹 AND 조건과 모바일 OR 조건을 각각 검증한다.
- 다른 목적·사용자·Session의 proof를 거부한다.
- proof 소비와 requested Link 생성의 원자성을 검증한다.
- 같은 key 재시도와 다른 payload 충돌에서 Link가 중복되지 않는다.
- 다른 사용자에 귀속된 번호로 계정이 병합되지 않는다.

## 연관 시퀀스

- [SCN.A.300-03 휴대폰 번호 교체](../../../80-sequence/A_300_auth/SCN_A_300_03_phone_replacement.md)
- 선행 API: `API.A.300-17`
- 후속 API: `API.A.300-22`, `API.A.300-23`
- 여러 참여자의 Mermaid 다이어그램은 시퀀스 문서에서 관리한다.

## 호환성과 변경 정책

- 응답에 선택 필드를 추가하는 변경은 하위 호환으로 처리한다.
- 전화번호 정규화 의미, proof 목적 또는 Link 상태 의미 변경은 새 API 버전에서 처리한다.
- 새 전화번호 입력 형식을 추가해도 기존 형식의 의미를 바꾸지 않는다.

## 확인 필요

- 전화번호 국가별 허용 형식과 정규화 오류의 공개 수준을 운영 정책으로 확정한다.
- requested Link 완료 기한과 멱등 레코드 보존 기간의 초기값을 확정한다.
