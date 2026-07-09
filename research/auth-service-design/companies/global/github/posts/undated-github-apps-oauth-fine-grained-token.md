# GitHub Apps, OAuth Apps, Fine-Grained Tokens

- 회사: GitHub
- 원문: https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps
- 원문: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: GitHub Apps, OAuth Apps, fine-grained PAT, short-lived token

## 원문 확인 내용

[GitHub Apps와 OAuth Apps 비교 문서](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps)는 GitHub Apps가 OAuth Apps보다 fine-grained permission, repository 접근 제어, short-lived token을 제공하기 때문에 일반적으로 선호된다고 설명한다. GitHub Apps는 사용자 대신 app installation으로 동작할 수 있어 automation에도 적합하다.

[GitHub PAT 관리 문서](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)는 fine-grained PAT가 resource owner, repository access, permission, expiration을 지정한다고 설명한다. 조직 owner가 approval 정책을 둘 수도 있다.

## 적용 아이디어

- DropMong의 외부 연동 token은 사용자 session token과 분리하고, 권한 범위와 만료시간을 명확히 둔다.
- 운영자/partner API token은 목적별로 쪼개고, 토큰마다 resource 범위를 제한한다.
- 장기 token은 정기 회전과 승인/철회 audit event를 가진다.

## 확인 필요

- [GitHub Apps 문서](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps)의 세부 rate limit 정책은 DropMong 로그인 설계와 직접 관련이 낮아 분석에서 보조 근거로만 사용한다.
