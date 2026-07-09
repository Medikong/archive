# Toss Zero Trust IAM/SSO

- 회사: Toss
- 원문: https://toss.tech/article/slash23-security
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: Zero Trust, IAM, SSO, MFA, device posture, RBAC/ABAC

## 원문 확인 내용

[Toss 기술 블로그의 Zero Trust 글](https://toss.tech/article/slash23-security)은 내부 업무용 application 접근을 IAM과 SSO에 연동했다고 설명한다. 기존 ID/password+OTP 방식에서 신뢰된 network, 회사 자산 여부, device 보안 수준 같은 조건을 함께 확인하는 구조로 확장한 점이 핵심이다.

같은 [Toss Zero Trust 글](https://toss.tech/article/slash23-security)은 인사 DB와 IAM을 연결해 직무 기반 접근제어를 구성하고, 퇴직이나 직무 변경에 따라 권한을 회수하거나 재할당하는 운영 방식을 언급한다.

## 적용 아이디어

- 고객 계정과 운영자 계정의 인증 강도를 분리한다.
- 운영자 콘솔은 조직/역할/기기 신뢰 조건을 일반 고객 API보다 먼저 도입한다.
- 권한 변경, 계정 잠금, MFA 등록/해제는 감사 로그의 필수 이벤트로 둔다.

## 확인 필요

- [Toss Zero Trust 글](https://toss.tech/article/slash23-security)은 고객 로그인 시스템이 아니라 내부 업무 시스템 인증 사례다. 고객 계정 설계에 그대로 적용하면 안 된다.
