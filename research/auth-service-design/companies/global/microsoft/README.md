# Microsoft

Microsoft Entra 문서를 기준으로 [Continuous Access Evaluation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation), [위험 기반 Conditional Access](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies), [MFA 배포 고려사항](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-getstarted)을 확인했다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| Continuous access evaluation in Microsoft Entra | https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation | posts/undated-microsoft-entra-cae-risk-mfa.md |
| Microsoft Entra ID Protection risk-based access policies | https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies | posts/undated-microsoft-entra-cae-risk-mfa.md |
| Deployment considerations for Microsoft Entra multifactor authentication | https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-getstarted | posts/undated-microsoft-entra-cae-risk-mfa.md |

## 원문 확인 내용

- [Continuous Access Evaluation 문서](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)는 token 만료시간만 기다리지 않고 critical event와 정책 평가로 접근을 재평가한다고 설명한다.
- [Microsoft Entra ID Protection 문서](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies)는 sign-in risk와 user risk를 Conditional Access에 전달하고, 위험 수준에 따라 MFA 요구나 차단을 적용할 수 있다고 설명한다.
- [Microsoft Entra MFA 배포 문서](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-getstarted)는 사용자를 너무 자주 재인증시키면 악성 prompt에도 습관적으로 응답할 위험이 있다고 경고한다.
- [Microsoft Entra MFA 배포 문서](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-getstarted)는 per-user MFA보다 Conditional Access 기반 MFA로 전환하는 방향을 권장한다.

## 설계 포인트

- access token 만료시간만으로 보안을 설명하면 부족하다. 계정 비활성화, 비밀번호 변경, 위험 탐지 같은 이벤트가 session/token에 반영되어야 한다.
- MFA는 모든 로그인에 고정 적용하기보다 위험도와 업무 중요도에 맞춰 정책화해야 한다.
- 보안 UX는 friction을 높이는 방향만으로 설계하면 실패한다. 사용자가 납득 가능한 순간에 추가 인증을 요구해야 한다.

## DropMong 적용 아이디어

- `session_risk_level`, `assurance_level`, `last_step_up_at`을 session/challenge 모델에 둔다.
- 계정 잠금, 비밀번호 변경, provider 연결 해제, refresh token 재사용 탐지 시 session invalidation 이벤트를 발행한다.
- MFA는 초기에는 고위험 작업 중심으로 적용하고, 로그인 전체 강제는 운영 데이터가 쌓인 뒤 검토한다.

## 확인 필요

- [Microsoft Entra 문서](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)는 enterprise IAM 제품 문서다. 소비자 커머스 서비스에 적용할 때는 기능 범위를 줄여야 한다.
