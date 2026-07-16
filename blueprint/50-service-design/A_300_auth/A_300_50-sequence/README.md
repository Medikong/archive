---
id: SCN.A.300
title: 인증 처리 시퀀스 인덱스
type: sequence-index
status: draft
tags: [sequence, scenario, auth, identity, session]
source: local
created: 2026-07-10
updated: 2026-07-15
use_case: UC.A.300
service_design: SD.A.300
---

# 인증 처리 시퀀스 인덱스

## 역할

Context 인증 내부의 여러 API와 Auth가 주도하는 처리 과정을 시나리오 단위로 관리한다. 여러 Context가 함께 참여하는 사용자 목적 시퀀스는 `80-sequence` 원장을 참조한다. 개별 Endpoint의 요청·응답은 API 문서가, Handler와 트랜잭션 책임은 서비스 문서가 기준이다.

외부 클라이언트의 Auth·User·업무 서비스 호출은 Istio Ingress Gateway를 거친다. 공개 Auth API에서는 TLS 종료, 라우팅, 요청 빈도 제한과 외부 내부용 헤더 제거만 담당한다. 보호 API에서는 access JWT의 서명·발급자·대상·만료와 Session 폐기 상태를 검증한 뒤 최소 사용자 컨텍스트를 만든다. Ingress는 로그인 성패, 업무 권한과 리소스 소유권을 판단하거나 여러 서비스 호출을 조정하지 않는다.

## 문서 목록

| Scenario ID | 처리 시퀀스 | 주요 API | 문서 |
| --- | --- | --- | --- |
| `SCN.A.01-01` | 회원가입과 자동 로그인 | `API.A.300-03~06`, `API.A.300-28`, `API.A.01-01` | [80-sequence 원장](../../../80-sequence/A_01_user/SCN_A_01_01_user_provisioning_auth_link.md) |
| `SCN.A.300-02` | 휴대폰 로그인과 refresh token 회전 | `API.A.300-08`, `API.A.300-09`, `API.A.300-14` | [상세](SCN_A_300_02_phone_signin_refresh.md) |
| `SCN.A.300-03` | 휴대폰 번호 교체 | `API.A.300-17`, `API.A.300-21~23` | [상세](SCN_A_300_03_phone_replacement.md) |
| `SCN.A.300-05` | 웹 JWT 로그인, 보호 API 인증, refresh와 로그아웃 | `API.A.300-01`, `API.A.300-07`, `API.A.300-14~16` | [상세](SCN_A_300_05_web_jwt_authentication.md) |
| `SCN.A.310-01` | 비밀번호 재설정 | `API.A.300-10~13` | [상세](SCN_A_310_01_password_reset.md) |

## 문서 경계

- 하나의 API 내부 처리만 설명할 때는 해당 API 문서에 기록한다.
- Auth 내부의 여러 API를 연결하는 처리 과정은 이 폴더에서 관리하고, 여러 Context를 연결하는 사용자 목적 시퀀스는 `80-sequence`에서 관리한다.
- 동일한 Mermaid 다이어그램을 API·서비스 문서에 복제하지 않고 해당 시퀀스 문서를 링크한다.
- 단계마다 관련 API, Command/Query, Aggregate와 Event 식별자를 가능한 범위에서 연결한다.
- 인증과 인가를 구분한다. Istio와 Auth는 credential의 유효성과 Principal만 확정하고 업무 인가와 리소스 소유권 판단은 각 업무 Context에 둔다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../../30-uc/UC_A_300_auth_member.md)
- [BC.A.300 인증 및 회원 Bounded Context](../../../40-event-storming-bounded-context/BC_A_300_auth_member.md)
- [SD.A.300 인증 서비스 상세 설계](../README.md)
- [SD.A.300.JWT JWT/JWKS/Istio 인증 처리 기준](../jwt-jwks-istio.md)
- [SD.A.30030 서비스 설계](../A_300_30-service/README.md)
- [SD.A.30040 API 공통 설계](../A_300_40-api/README.md)
- [SCN.A.01 사용자 서비스 처리 시퀀스](../../A_01_user/A_01_50-sequence/README.md)

가입 완료는 프론트엔드가 Auth와 User의 공개 API를 차례로 호출해 처리한다. Event 기반 사용자 연결과 상태 polling은 사용하지 않는다.

## 확인 필요

- 운영자 정책 변경과 수동 인증 처리도 독립 시퀀스가 필요한지 운영 절차가 확정된 뒤 판단한다.
- 외부 Provider 로그인 도입 시 Provider별 시퀀스를 이 인덱스에 추가한다.
