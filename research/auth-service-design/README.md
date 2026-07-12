# Auth Service Design Research

국내/해외 기업의 인증 서비스 설계 사례를 공개 자료 기준으로 조사한 문서 묶음이다.

목표는 실제 회사 기술 블로그, 공식 개발자 문서, 공개 아키텍처 글에서 인증 서비스의 반복 패턴을 모으고, DropMong 인증/회원 설계에 참고할 수 있는 선택지를 정리하는 것이다. 이 문서 묶음은 조사와 분석을 위한 공간이며, 실제 적용 결정은 별도 설계 문서나 ADR에서 다룬다.

## 조사 결과 요약

- 국내 자료는 [Kakao](https://developers.kakao.com/docs/en/kakaologin/common), [Naver](https://developers.naver.com/docs/login/devguide/devguide.md), [Toss](https://toss.tech/article/slash23-security), [LINE](https://developers.line.biz/en/docs/line-login/integrate-line-login/)의 공식 문서와 기술 블로그를 중심으로 확인했다.
- 해외 자료는 [Google](https://developers.google.com/identity/openid-connect/openid-connect), [Microsoft](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation), [Netflix](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602), [Uber](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/), [GitHub](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps), [Stripe](https://docs.stripe.com/keys/restricted-api-keys), [Auth0](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)의 공식 문서와 엔지니어링 블로그를 중심으로 확인했다.
- [Kakao](https://developers.kakao.com/docs/en/kakaologin/rest-api), [LINE](https://developers.line.biz/en/docs/line-login/verify-id-token/), [Google](https://developers.google.com/identity/openid-connect/openid-connect), [Auth0](https://auth0.com/docs/manage-users/user-accounts/user-account-linking) 자료는 외부 provider identity와 내부 회원 계정의 매핑, OIDC ID token 검증, 토큰 폐기, 계정 연결 위험을 보는 데 유용했다.
- [Netflix](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602)와 [Uber](https://www.uber.com/us/en/blog/our-journey-adopting-spiffe-spire/) 자료는 gateway/edge나 공통 인증 라이브러리에서 인증을 중앙 처리하고 downstream에는 검증된 identity 구조만 넘기는 패턴을 보여준다.
- [Auth0](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation), [Microsoft](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies), [Toss](https://toss.tech/article/slash23-security), [GitHub](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) 자료는 refresh token rotation, token family revocation, 위험 기반 MFA, device posture, workload identity, 감사 로그와 추적 가능성을 우선 고려해야 한다는 점을 보여준다.

## 읽는 순서

| 문서 | 용도 |
| --- | --- |
| [sources.md](sources.md) | 수집 후보와 저장한 출처 목록 |
| [taxonomy.md](taxonomy.md) | 인증 서비스 설계 관점 분류 기준 |
| [companies/domestic/README.md](companies/domestic/README.md) | 국내 기업별 조사 인덱스 |
| [companies/global/README.md](companies/global/README.md) | 해외 기업별 조사 인덱스 |
| [analysis/README.md](analysis/README.md) | 주제별 2차 분석 인덱스 |

## 빠른 결론

| 주제 | 확인한 패턴 | DropMong 설계 메모 |
| --- | --- | --- |
| 인증/회원 경계 | [외부 IdP 인증 결과는 내부 회원 가입/로그인 판단과 분리된다](https://developers.kakao.com/docs/en/kakaologin/rest-api). | `auth-service`는 credential, provider identity, session/token을 다루고 `member-service`는 프로필과 회원 상태를 관리한다. |
| 토큰/세션 | [access token은 짧게, refresh token은 회전/폐기/재사용 탐지를 둔다](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation). | refresh token family와 device session을 DB 모델에 반영한다. |
| OAuth/OIDC | [provider `sub`와 내부 `member_id`를 직접 동일시하지 않는다](https://openid.net/specs/openid-connect-core-1_0.html). | `external_identity(provider, subject)` 고유 제약을 둔다. |
| MFA/위험 기반 인증 | [모든 요청에 MFA를 강제하기보다 위험 신호나 민감 작업에서 step-up을 건다](https://learn.microsoft.com/en-us/entra/id-protection/concept-identity-protection-policies). | 결제수단 변경, 고가 구매, 계정 연결, 휴면 복구에 추가 인증을 둔다. |
| 내부 서비스 인증 | [사용자 토큰과 서비스 토큰을 분리하고, edge/gateway가 검증된 사용자 맥락을 전달한다](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602). | 내부 호출에는 workload identity 또는 mTLS를 장기 목표로 두고, 초기에는 gateway-signed internal claim을 검토한다. |
| 운영/보안 | [인증 이벤트는 추적 가능해야 하고, 비정상 시 rate limit과 revocation이 작동해야 한다](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation). | 로그인 실패, 토큰 발급/회전/폐기, 계정 연결, 권한 변경을 감사 로그로 남긴다. |

## 폴더 구조

```text
auth-service-design/
  README.md
  sources.md
  taxonomy.md
  companies/
    domestic/
      README.md
      {company}/
        README.md
        posts/
    global/
      README.md
      {company}/
        README.md
        posts/
  analysis/
    README.md
    01-patterns.md
    02-session-token.md
    03-oauth-oidc-social-login.md
    04-mfa-risk-auth.md
    05-account-linking.md
    06-internal-service-auth.md
    07-operations-security.md
    08-dropmong-implications.md
  templates/
    company-profile.md
    source-note.md
    post-scrap.md
```

## 조사 단위

- 회사별 `README.md`: 회사의 인증/회원/보안 설계 특징을 요약한다.
- 회사별 출처 목록은 각 회사 `README.md`의 `확인한 공개 자료` 표와 루트 [sources.md](sources.md)에 함께 둔다.
- `posts/`: 공개 포스트를 스크랩한 본문을 둔다.
- `assets/`: 원문 도식, 캡처, 참고 이미지가 필요할 때만 둔다.
- `analysis/`: 여러 회사 자료를 주제별로 다시 묶는다.

## 현재 범위

- 국내/해외 기업의 공개 인증 서비스 설계 글과 공식 개발자 문서를 조사한다.
- 로그인, 세션, 토큰, OAuth/OIDC, MFA, 계정 연결, 내부 서비스 인증, 감사 로그, 운영/보안 관점을 중심으로 본다.
- 공개 자료에 없는 내부 구현은 추정하지 않는다. 필요하면 `추정` 또는 `확인 필요`로 남긴다.
