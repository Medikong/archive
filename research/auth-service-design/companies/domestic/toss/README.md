# Toss

Toss 자료는 두 갈래로 보았다. 기술 블로그의 [Zero Trust 글](https://toss.tech/article/slash23-security)은 내부 업무 시스템 인증과 권한 운영 관점이고, 앱인토스 개발자센터의 [토스 인증 문서](https://developers-apps-in-toss.toss.im/tossauth/develop.html)는 본인확인 API 관점이다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| 금융사 최초의 Zero Trust 아키텍처 도입기 | https://toss.tech/article/slash23-security | posts/undated-toss-zero-trust-iam-sso.md |
| 토스 인증 | https://developers-apps-in-toss.toss.im/tossauth/develop.html | posts/undated-toss-cert-api.md |

## 원문 확인 내용

- [Toss Zero Trust 글](https://toss.tech/article/slash23-security)은 내부 application 로그인을 IAM/SSO와 연동하고, ID/password+OTP뿐 아니라 신뢰된 network, 회사 자산 여부, device 보안 수준을 함께 검증한다고 설명한다.
- [Toss Zero Trust 글](https://toss.tech/article/slash23-security)은 인사 DB와 IAM을 연동해 직무 기반 접근제어와 권한 회수를 운영한다고 설명한다.
- [토스 인증 API 문서](https://developers-apps-in-toss.toss.im/tossauth/develop.html)는 본인확인 요청에서 `txId`를 발급하고, 사용자가 인증을 완료하면 서버 간 결과 조회로 최종 확인하도록 설계되어 있다.
- [토스 인증 API 문서](https://developers-apps-in-toss.toss.im/tossauth/develop.html)는 개인정보 기반 인증 요청과 결과 조회 모두에서 session key를 매 요청 새로 생성하라고 안내한다.

## 설계 포인트

- [Toss Zero Trust 글](https://toss.tech/article/slash23-security)을 보면 금융권 인증은 로그인 UX만의 문제가 아니라 device posture, 업무 시스템 접근제어, 권한 회수, 감사 가능성을 함께 다룬다.
- 본인확인 API는 사용자 앱에서 완료 신호를 받더라도 서버 간 결과 조회를 통해 최종 판단해야 한다.
- `REQUESTED`, `IN_PROGRESS`, `COMPLETED`, `EXPIRED` 같은 인증 transaction 상태는 결제 전 step-up 인증에도 적용할 수 있다.

## DropMong 적용 아이디어

- 위험 기반 인증을 바로 전체 로그인에 강제하기보다 고위험 작업에 `auth_challenge` transaction으로 도입한다.
- `tx_id`, `challenge_type`, `status`, `expires_at`, `verified_at`, `attempt_count`를 가진 challenge 테이블을 둔다.
- 운영자/관리자 계정은 일반 고객 계정보다 device posture, MFA, RBAC/ABAC를 먼저 적용한다.

## 확인 필요

- [Toss Zero Trust 글](https://toss.tech/article/slash23-security)과 [토스 인증 API 문서](https://developers-apps-in-toss.toss.im/tossauth/develop.html)에서는 Toss 고객 서비스 로그인 내부 구조, refresh token 모델, 계정 연결 정책을 확인하지 못했다.
