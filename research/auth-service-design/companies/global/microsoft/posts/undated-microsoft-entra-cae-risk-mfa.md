# Microsoft Entra CAE, Risk, MFA

- 회사: Microsoft
- 원문: https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation
- 원문: https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies
- 원문: https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-getstarted
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: Continuous Access Evaluation, risk-based policy, MFA rollout

## 원문 확인 내용

[Continuous Access Evaluation 문서](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)는 critical event와 정책 평가를 통해 접근을 재평가한다고 설명한다. 기본 access token lifetime만으로 접근 종료를 기다리는 방식보다 정책 반영이 빠르다.

[ID Protection risk-based policy 문서](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies)는 sign-in risk와 user risk를 조건으로 삼고, 위험도가 medium/high일 때 MFA 같은 추가 조치를 요구할 수 있다고 설명한다.

[MFA 배포 문서](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-mfa-getstarted)는 사용자에게 너무 자주 재인증을 요구하면 사용자가 악성 인증 prompt에도 습관적으로 응답할 수 있다고 경고한다.

## 적용 아이디어

- session/token은 만료시간 외에도 `revoked_reason`과 `risk_event_id`를 가져야 한다.
- 위험 기반 MFA는 `risk_score >= threshold` 또는 `sensitive_action`에서만 발동하도록 시작한다.
- MFA prompt 남발을 막기 위해 challenge 성공 후 짧은 `assurance window`를 둔다.

## 확인 필요

- [Microsoft Entra ID Protection 문서](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies)는 risk-based policy 동작을 설명하지만, risk score 산정 알고리즘은 제품 내부 구현이므로 공개 문서만으로 재현할 수 없다.
