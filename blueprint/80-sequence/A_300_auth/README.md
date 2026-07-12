---
id: SCN.A.300
title: 인증 처리 시퀀스 인덱스
type: sequence-index
status: draft
tags: [sequence, scenario, auth, identity, session]
source: local
created: 2026-07-10
updated: 2026-07-10
use_case: UC.A.300
service_design: SD.A.300
---

# 인증 처리 시퀀스 인덱스

## 역할

Context 인증의 여러 API와 외부 Context가 함께 참여하는 처리 과정을 시나리오 단위로 관리한다. 개별 Endpoint의 요청·응답은 API 문서가, Handler와 트랜잭션 책임은 서비스 문서가 기준이며, 이 폴더는 참여자 사이의 호출 순서와 성공·실패 조건을 연결한다.

## 문서 목록

| Scenario ID | 처리 시퀀스 | 주요 API | 문서 |
| --- | --- | --- | --- |
| `SCN.A.300-01` | 이메일 회원가입과 자동 로그인 | `API.A.300-03~06`, `API.A.300-28` | [상세](SCN_A_300_01_email_registration.md) |
| `SCN.A.300-02` | 휴대폰 로그인과 refresh token 회전 | `API.A.300-08`, `API.A.300-09`, `API.A.300-14` | [상세](SCN_A_300_02_phone_signin_refresh.md) |
| `SCN.A.300-03` | 휴대폰 번호 교체 | `API.A.300-17`, `API.A.300-21~23` | [상세](SCN_A_300_03_phone_replacement.md) |
| `SCN.A.310-01` | 비밀번호 재설정 | `API.A.300-10~13` | [상세](SCN_A_310_01_password_reset.md) |

## 문서 경계

- 하나의 API 내부 처리만 설명할 때는 해당 API 문서에 기록한다.
- 여러 API 또는 Context를 연결하는 처리 과정은 이 폴더에서 관리한다.
- 동일한 Mermaid 다이어그램을 API·서비스 문서에 복제하지 않고 해당 시퀀스 문서를 링크한다.
- 단계마다 관련 API, Command/Query, Aggregate와 Event 식별자를 가능한 범위에서 연결한다.

## 연관 문서

- [REQ.A.05 인증 및 회원 요구사항](../../00-requirements/REQ_A_05_auth_member.md)
- [UC.A.300 인증 및 회원 사용자 목표](../../30-uc/UC_A_300_auth_member.md)
- [BC.A.300 인증 및 회원 Bounded Context](../../40-event-storming-bounded-context/BC_A_300_auth_member.md)
- [SD.A.300 인증 서비스 상세 설계](../../50-service-design/A_300_auth/README.md)
- [SD.A.30030 서비스 설계](../../50-service-design/A_300_auth/A_300_30-service/README.md)
- [SD.A.30040 API 공통 계약](../../50-service-design/A_300_auth/A_300_40-api/README.md)

## 확인 필요

- 운영자 정책 변경과 수동 인증 처리도 독립 시퀀스가 필요한지 운영 절차가 확정된 뒤 판단한다.
- 외부 Provider 로그인 도입 시 Provider별 시퀀스를 이 인덱스에 추가한다.
