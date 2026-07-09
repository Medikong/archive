# GitHub

GitHub 자료는 개발자 플랫폼 인증과 권한 범위 축소 관점으로 조사했다. [OAuth Apps와 GitHub Apps](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps), [fine-grained PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)를 비교해 내부 서비스/API 인증 원칙을 참고한다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| Differences between GitHub Apps and OAuth apps | https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps | posts/undated-github-apps-oauth-fine-grained-token.md |
| Managing your personal access tokens | https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens | posts/undated-github-apps-oauth-fine-grained-token.md |

## 원문 확인 내용

- [GitHub Apps와 OAuth Apps 비교 문서](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps)는 GitHub Apps가 OAuth Apps보다 fine-grained permission, repository 접근 제어, short-lived token 측면에서 선호된다고 설명한다.
- [GitHub Apps와 OAuth Apps 비교 문서](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps)는 GitHub Apps가 사용자 없이도 독립적으로 동작할 수 있어 조직 automation에 적합하고, installation access token의 rate limit도 확장된다고 설명한다.
- [GitHub PAT 관리 문서](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)는 fine-grained PAT에서 resource owner, repository access, permission, expiration을 지정할 수 있고, 조직 owner approval을 요구할 수 있다고 설명한다.
- [GitHub PAT 관리 문서](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)는 classic PAT가 더 넓은 접근 권한을 줄 수 있어 보안상 주의가 필요하다고 설명한다.

## 설계 포인트

- 사용자 위임 token과 application/installation token을 분리하면 자동화와 사용자 권한을 더 명확히 나눌 수 있다.
- token은 "소유자가 가진 모든 권한"을 기본으로 주지 말고, resource와 permission을 줄여야 한다.
- token 만료와 조직 승인 정책은 보안 운영 기능으로 제품 UX에 들어가야 한다.

## DropMong 적용 아이디어

- 내부 운영용 API token은 customer access token과 다른 issuer/audience를 가진다.
- partner/integration token은 `scope`, `resource_owner`, `expires_at`, `approved_by`를 가진 별도 모델로 설계한다.
- 관리자 API는 short-lived token 또는 step-up challenge를 요구한다.

## 확인 필요

- [GitHub Apps 문서](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps)는 consumer login보다 developer platform authorization에 가깝다. 회원 로그인 모델에 직접 대입하지 않는다.
