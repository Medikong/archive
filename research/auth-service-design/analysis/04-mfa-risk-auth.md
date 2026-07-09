# MFA와 위험 기반 인증

## 질문

MFA와 위험 기반 인증은 언제 필요한가?

## 원문에서 확인한 내용

- [Toss Zero Trust 글은 내부 업무 application 로그인에서 IAM/SSO, MFA, 신뢰된 network, 회사 자산 여부, device 보안 수준을 함께 검증한다고 설명한다](https://toss.tech/article/slash23-security).
- [Google Workspace security challenge는 의심 로그인과 민감 작업에서 추가 확인을 요구한다](https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges).
- [Microsoft Entra risk-based policy는 sign-in risk와 user risk에 따라 MFA나 차단을 적용할 수 있다고 설명한다](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies).
- [Microsoft MFA 배포 문서는 사용자를 너무 자주 재인증시키면 악성 prompt에도 습관적으로 응답할 수 있다고 경고한다](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-getstarted).
- [Netflix는 의심스러운 연결에 MFA를 선택적으로 도입하는 방향을 설명한다](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602).
- [Toss 인증 API는 본인확인 작업을 `txId` 중심 transaction으로 처리하고, 완료 후 서버 간 결과 조회로 최종 확인한다](https://developers-apps-in-toss.toss.im/tossauth/develop.html).

## MFA를 요구할 후보 조건

| 조건 | 예시 | 권장 대응 |
| --- | --- | --- |
| 로그인 위험 | 새 기기, 이상 IP, 비정상 실패 횟수 | 추가 인증 또는 일시 차단 |
| 민감 작업 | 비밀번호 변경, 소셜 계정 연결, 결제수단 변경 | 현재 credential 재확인 또는 MFA |
| 고위험 구매 | 고가 상품, 자동화 의심, 빠른 반복 시도 | step-up challenge와 rate limit |
| 운영자 접근 | 관리자 콘솔, 권한 변경, 수동 보정 | MFA 필수, device trust 확인 |
| 계정 복구 | 휴면 복구, 전화번호 변경, 이메일 변경 | 강한 본인확인과 지연 처리 |

## DropMong `auth_challenges` 모델

| 필드 | 설명 |
| --- | --- |
| `id` | challenge id |
| `member_id` | 대상 회원 |
| `session_id` | challenge를 시작한 session |
| `challenge_type` | password, otp, passkey, provider_reauth, phone_cert |
| `reason` | account_linking, high_value_purchase, password_change 등 |
| `status` | requested, in_progress, verified, expired, failed |
| `risk_level` | low, medium, high |
| `expires_at`, `verified_at` | 만료/성공 시각 |
| `attempt_count` | 재시도 횟수 |
| `result_ref` | 외부 본인확인 결과 id 또는 내부 검증 id |

## 적용 원칙

- 로그인 전체에 MFA를 고정 적용하기보다 민감 작업과 위험 신호에서 step-up challenge를 먼저 도입한다.
- challenge 성공 결과는 짧은 시간만 재사용한다. 예를 들어 계정 연결용 challenge 성공은 구매 승인에는 쓰지 않는다.
- MFA prompt가 자주 뜨면 사용자가 무비판적으로 승인할 수 있으므로, challenge reason을 명확히 남기고 알림 문구도 구체화한다.
- 운영자 계정은 고객 계정보다 강한 정책을 먼저 적용한다.

## 확인 필요

- DropMong에 SMS/이메일 OTP, passkey, 외부 본인확인 중 무엇을 우선 도입할지는 제품 비용과 사용자 경험을 함께 봐야 한다.
- 위험 점수 산정은 초기에는 rule 기반으로 시작하고, ML 기반 탐지는 운영 데이터가 쌓인 뒤 검토한다.

## 출처

| 회사 | 출처 | 저장 위치 | 확인한 내용 |
| --- | --- | --- | --- |
| Toss | https://toss.tech/article/slash23-security | companies/domestic/toss/posts/undated-toss-zero-trust-iam-sso.md | IAM/SSO/MFA와 device posture 검증 |
| Toss | https://developers-apps-in-toss.toss.im/tossauth/develop.html | companies/domestic/toss/posts/undated-toss-cert-api.md | `txId` 기반 본인확인 transaction과 서버 간 결과 조회 |
| Google | https://knowledge.workspace.google.com/admin/security/protect-google-workspace-accounts-with-security-challenges | companies/global/google/posts/undated-google-openid-connect-iap.md | 의심 로그인과 민감 작업 security challenge |
| Microsoft | https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies | companies/global/microsoft/posts/undated-microsoft-entra-cae-risk-mfa.md | sign-in risk/user risk 기반 Conditional Access |
| Netflix | https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602 | companies/global/netflix/posts/undated-netflix-edge-authentication-passport.md | 의심 연결에 선택적 MFA 도입 방향 |
