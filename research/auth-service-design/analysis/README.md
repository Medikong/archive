# Analysis

회사별로 수집한 인증 서비스 설계 자료를 주제별로 다시 정리하는 영역이다.

## 읽는 기준

- `원문에서 확인한 내용`은 공개 자료에서 직접 확인한 사실만 담는다.
- `DropMong 적용 아이디어`는 공개 자료를 바탕으로 DropMong에 맞게 해석한 설계안이다.
- `확인 필요`는 공개 자료만으로 단정할 수 없는 내부 구현 또는 운영 정책이다.

## 분석 문서

| 문서 | 질문 | 핵심 결론 |
| --- | --- | --- |
| [01-patterns.md](01-patterns.md) | 여러 회사에서 반복되는 인증 서비스 구조는 무엇인가? | 외부 인증, 내부 회원, 토큰/session, 감사 로그는 서로 다른 책임으로 나뉜다. |
| [02-session-token.md](02-session-token.md) | 세션, access token, refresh token은 어떻게 나뉘는가? | access token은 짧게, refresh token은 family/rotation/revocation 단위로 관리한다. |
| [03-oauth-oidc-social-login.md](03-oauth-oidc-social-login.md) | 외부 IdP와 소셜 로그인은 내부 계정과 어떻게 연결되는가? | provider `sub`와 내부 `member_id`를 분리하고, ID token은 서버에서 검증한다. |
| [04-mfa-risk-auth.md](04-mfa-risk-auth.md) | MFA와 위험 기반 인증은 언제 필요한가? | 모든 로그인보다 민감 작업과 위험 신호에서 step-up challenge를 요구한다. |
| [05-account-linking.md](05-account-linking.md) | 여러 인증 수단과 계정 병합은 어떻게 다루는가? | 동일 email은 연결 제안 근거일 뿐이며, 연결 전 양쪽 계정 재인증이 필요하다. |
| [06-internal-service-auth.md](06-internal-service-auth.md) | 내부 서비스 간 인증은 어떤 방식으로 설계되는가? | 사용자 인증과 workload/service 인증을 분리하고, gateway가 검증된 identity context를 전달한다. |
| [07-operations-security.md](07-operations-security.md) | 키 회전, 감사 로그, rate limit, abuse 대응은 어떻게 운영되는가? | 인증 이벤트를 감사 로그로 남기고 token/key 폐기와 rate limit을 운영 기능으로 둔다. |
| [08-dropmong-implications.md](08-dropmong-implications.md) | DropMong 인증/회원 설계에 적용할 점은 무엇인가? | `auth-service`와 `member-service`를 분리하고 token/session/challenge/audit 모델을 우선 설계한다. |

## 작성 기준

- 회사별 원문 링크와 저장 위치를 함께 남긴다.
- 공개 자료에서 확인한 내용과 적용 아이디어를 구분한다.
- 특정 기업 사례 하나만으로 일반화하지 않는다.
