---
id: API.A.300-30
title: 가상 인증 메시지 조회 API
type: service-design-api-endpoint
status: draft
service_design: SD.A.300
api_design: SD.A.30040
domain_model: SD.A.30010
persistence: SD.A.30020
service: SD.A.30030
updated: 2026-07-10
---

# API.A.300-30 가상 인증 메시지 조회

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /api/v1/dev/auth/verification-messages/{challengeId}` |
| operationId | `getVirtualVerificationMessage` |
| 역할 | 개발·테스트 환경에서 가상 Adapter가 전달한 이메일·SMS 인증번호를 조회한다. |
| API 유형 | Query |
| 인증 | 개발 접근 token과 Challenge 소유 credential의 AND 조건 |
| 노출 범위 | `local`, `test`, 접근이 제한된 `demo` |
| 멱등성 | GET, 같은 projection version에서는 같은 결과 |
| 캐시 | 성공·오류 응답 모두 `no-store` |
| 호환성 | 개발 API bundle에만 포함하고 운영 Route에는 등록하지 않음 |

## HTTP 계약 원장

- 개발 OpenAPI 진입 문서: [openapi/dev.openapi.yaml](openapi/dev.openapi.yaml)
- 이 Endpoint의 Path Item: [openapi/paths/API_A_300_30_get_virtual_verification_message.yaml](openapi/paths/API_A_300_30_get_virtual_verification_message.yaml)
- 공통 component: [schemas](openapi/components/schemas.yaml), [parameters](openapi/components/parameters.yaml), [headers](openapi/components/headers.yaml), [responses](openapi/components/responses.yaml), [security schemes](openapi/components/security-schemes.yaml)

요청 header, 소유 인증 조합, 공통 `VerificationCode`를 쓰는 `ready`·`pending` 응답 schema, 상태 코드, 오류 body와 wire 예시는 OpenAPI를 기준으로 한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [SD.A.30010 인증 도메인 모델](../A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- [SD.A.30020 인증 영속성 설계](../A_300_20-persistence/README.md)
- [SD.A.30030 인증 서비스 설계](../A_300_30-service/README.md)
- [SD.A.30040 API 공통 계약](README.md)

## 책임과 경계

- 가상 Adapter가 만든 단기 메시지 projection의 현재 전달 상태와 인증번호를 허용된 개발 주체에게만 보여준다.
- 가상 이메일과 가상 SMS는 모두 사용자가 직접 입력하는 code 방식을 사용한다.
- 인증번호는 운영 Challenge와 같은 공통 `VerificationCode`의 6자리 숫자 형식을 사용한다.
- 조회는 Challenge를 검증·소비하거나 Identity를 verified로 바꾸지 않는다.
- 인증번호 자동 입력, Challenge 검증 API 호출, 이메일·휴대폰 번호 검색과 최근 메시지 목록 조회는 제공하지 않는다.
- 운영 환경에서는 Gateway 차단에만 의존하지 않고 Route와 virtual delivery 구현을 등록하지 않는다.

## 보안과 개인정보

- 개발 접근 token과 웹 사전 인증 cookie, 사용자 Session, 모바일 flow token 또는 access token 중 하나의 Challenge 소유 credential을 함께 검증한다.
- `challengeId`, 소유 credential과 Challenge binding이 모두 일치해야 한다.
- raw destination은 반환하지 않고 인증 화면에 필요한 마스킹 값만 제공한다.
- 인증번호, destination, 개발 접근 token, flow token과 cookie 원문을 로그·trace·metric label에 넣지 않는다.
- 응답 body sampling, 브라우저·CDN 캐시와 검색용 색인을 비활성화한다.
- 공유 demo에서는 허용된 개발 네트워크 또는 test client identity를 추가로 요구하고 가상 인증임을 화면에 표시한다.

## 처리 규칙

1. 개발 환경과 개발 접근 token을 검증한다.
2. 현재 credential이 Challenge 소유 binding과 일치하는지 확인한다.
3. delivery mode가 `virtual`이고 channel이 `email_code` 또는 `sms_code`인지 확인한다.
4. 개발·테스트 profile 전용 `auth_virtual_verification_messages` projection을 읽는다.
5. 아직 Adapter 처리가 끝나지 않았으면 code 없이 대기 결과를 반환한다.
6. 준비된 메시지는 메모리에서만 복호화해 code와 마스킹 수신지를 반환한다.

## 상태 변경과 트랜잭션

- 이 Query는 VerificationChallenge 상태, 검증 실패 횟수와 소비 시각을 변경하지 않는다.
- `secret_hash`를 역산하지 않고 별도 단기 메시지 projection만 읽는다.
- projection은 `challenge_id`, channel, 암호화된 code, key ID, 마스킹 수신지, 상태, Challenge version, 만료·생성·폐기 시각만 저장한다.
- code 원문은 envelope encryption으로 보관하고 응답 생성 중에만 복호화한다.
- Challenge가 verified, expired, failed 또는 revoked로 종료되면 같은 transaction에서 projection을 `destroyed`로 바꾸고 암호화된 code를 지우며 `destroyed_at`을 기록한다.
- 폐기된 행은 감사에 필요한 비밀 없는 최소 정보만 TTL 동안 유지한 뒤 정리한다.
- 운영 설정에서 virtual delivery가 감지되면 애플리케이션 시작을 실패시킨다.

## 멱등성과 동시성

- 같은 Challenge와 소유 credential의 반복 GET은 같은 projection version에서 같은 상태를 반환한다.
- 조회 횟수는 Challenge 검증 시도 횟수나 code 사용 가능 횟수에 영향을 주지 않는다.
- Adapter 저장과 조회가 경합하면 준비 전에는 대기 결과를 반환하고 다음 조회에서 준비된 결과를 반환한다.
- Challenge 종료와 조회가 경합하면 종료 상태와 projection 폐기를 우선하며 code를 반환하지 않는다.
- 조회 rate limit은 개발 token, 소유 credential과 설치 범위를 조합해 적용한다.

## 예외와 복구 규칙

정확한 HTTP 상태, error code와 ProblemDetails 예시는 OpenAPI를 기준으로 한다.

| 업무 조건 | 공개 원칙 | 클라이언트 복구 |
| --- | --- | --- |
| 개발 접근 또는 소유 binding 검증 실패 | Challenge와 메시지 존재 여부를 구분하지 않는다. | 올바른 테스트 환경과 인증 컨텍스트에서 다시 시작한다. |
| Challenge 종료 또는 code 폐기 완료 | 종료 원인과 code 원문을 공개하지 않는다. | 새 Challenge를 발급한다. |
| 조회 제한 초과 | 내부 제한 수치와 다른 사용자의 활동을 노출하지 않는다. | `Retry-After` 이후 다시 조회한다. |
| projection 저장소 장애 | 성공 형태나 임의 code로 대체하지 않는다. | 같은 credential로 재시도한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Query Handler | `GetVirtualVerificationMessageHandler` |
| Domain | `VerificationChallenge` 상태와 소유 binding |
| Port / Adapter | `VerificationDeliveryPort`, `VirtualVerificationAdapter` |
| Read Model | 개발·테스트 전용 `auth_virtual_verification_messages` 단기 projection |
| 후속 API | `API.A.300-05`, `API.A.300-09`, `API.A.300-12`, `API.A.300-20`, `API.A.300-23` |

가상 메시지 projection은 테스트 편의를 위한 조회 모델이며 VerificationChallenge Aggregate 또는 Identity 소유 확인 결과를 대신하지 않는다.

## 관측성과 운영

- 로그에는 환경, channel, 결과와 request ID만 기록하고 Challenge ID와 code를 제외한다.
- `auth_virtual_verification_message_query_total{channel,result}`로 조회 결과를 집계한다.
- trace에는 개발 환경 검증, 소유 binding 검증과 projection 조회 구간만 남긴다.
- audit에는 code 없이 조회 주체 유형, Challenge purpose, 결과와 시각을 기록한다.
- 운영 배포 검증은 공개 bundle에 `/api/v1/dev/` 경로가 있거나 virtual Adapter bean이 등록되면 실패해야 한다.

## 검증 항목

- Adapter 처리 전 응답에는 code와 destination이 없다.
- ready 응답의 code는 공통 `VerificationCode` 6자리 숫자 schema를 통과한다.
- 같은 Challenge와 소유 컨텍스트의 반복 조회는 TTL 안에서 같은 code를 반환한다.
- 다른 flow token, cookie 또는 Session은 메시지 존재 여부를 확인할 수 없다.
- 검증되거나 만료된 Challenge의 code는 조회할 수 없다.
- 조회 반복이 Challenge 상태와 검증 실패 횟수를 바꾸지 않는다.
- 이메일·휴대폰 번호로 검색하거나 메시지 목록을 조회할 수 없다.
- 응답·로그·trace에 raw destination과 credential 원문이 남지 않는다.
- 운영 환경에는 Route와 virtual delivery 구현이 없다.

## 연관 시퀀스

- 계획 문서: `SCN.A.300-04 가상 인증 메시지 전달과 조회`
- 관련 API: `API.A.300-04`, `API.A.300-08`, `API.A.300-11`, `API.A.300-19`, `API.A.300-22`, `API.A.300-30`
- 여러 참여자의 Mermaid 다이어그램은 [인증 시퀀스 폴더](../../../80-sequence/A_300_auth/README.md)에서 관리한다.

## 호환성과 변경 정책

- 개발 API는 운영 공개 API와 별도 bundle·Gateway route·코드 생성 입력을 사용한다.
- `ready` 또는 `pending` variant에 optional 필드를 추가할 때도 code 노출 범위와 캐시 정책을 다시 검토한다.
- 공통 `VerificationCode` 형식 변경은 Challenge 발급·검증 API와 Virtual Adapter를 같은 버전에서 변경한다.
- 개발 API를 제거해도 운영 공개 API 버전에는 breaking change로 계산하지 않는다.

## 확인 필요

- `SCN.A.300-04` 시퀀스 문서를 추가한다.
