# Domestic Companies

국내 기업의 인증 서비스 설계 사례를 회사별로 정리한다.

## 회사 인덱스

| 회사 | 조사 상태 | 주요 주제 | 시작 문서 |
| --- | --- | --- | --- |
| Kakao | 분석 | Kakao Login, OAuth 2.0, OIDC, service user id, token 만료 | [kakao/README.md](kakao/README.md) |
| Naver | 분석 | Naver Login, OAuth 2.0, token revocation, 연동 해제 | [naver/README.md](naver/README.md) |
| Toss | 분석 | Zero Trust, IAM/SSO/MFA, 본인확인 API, txId/sessionKey | [toss/README.md](toss/README.md) |
| LINE | 분석 | LINE Login, OIDC, ID token 검증, callback URL | [line/README.md](line/README.md) |
| 당근 | 보류 | 이번 조사에서는 인증 설계 직접 자료를 우선하지 않아 제외 | - |

## 이번 조사에서 얻은 국내 사례 포인트

- [Kakao](https://developers.kakao.com/docs/en/kakaologin/rest-api)와 [Naver](https://developers.naver.com/docs/login/devguide/devguide.md)는 외부 로그인 provider가 인증과 동의를 처리하고, 서비스 서버가 내부 회원 여부를 판단하는 구조를 명확히 보여준다.
- [Toss Zero Trust 글](https://toss.tech/article/slash23-security)과 [토스 인증 API 문서](https://developers-apps-in-toss.toss.im/tossauth/develop.html)는 고객 로그인보다는 내부 업무 시스템과 본인확인 API 관점이 강하지만, MFA, device posture, 결과 조회 제한, 서버 간 조회 같은 운영 보안 단서가 많다.
- [LINE ID token 문서](https://developers.line.biz/en/docs/line-login/verify-id-token/)는 OIDC ID token을 서비스가 그대로 믿지 말고 서명, issuer, audience, expiry를 검증해야 한다는 점을 분명히 보여준다.
- 국내 소셜 로그인 자료만으로는 내부 인증 DB 전체 모델을 단정하기 어렵다. `external_identity`와 `member`를 분리하는 적용 아이디어는 자료 조합에 기반한 설계 해석이다.

## 회사별 폴더 규칙

```text
{company}/
  README.md
  posts/
```

- 회사 폴더명은 소문자 kebab-case로 쓴다.
- `README.md`는 회사별 요약과 주요 설계 포인트를 담는다.
- `posts/`에는 포스트별 스크랩을 둔다.
- `assets/`는 원문 도식, 캡처, 참고 이미지가 필요할 때만 둔다.
