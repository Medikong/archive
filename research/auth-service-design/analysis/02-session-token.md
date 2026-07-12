# 세션과 토큰

## 질문

세션, access token, refresh token은 어떻게 나뉘는가?

## 원문에서 확인한 내용

- [Kakao는 REST API 기준 access token을 6시간, refresh token을 2개월로 설명하고, refresh token 갱신 시 이전 refresh token을 폐기한다고 설명한다](https://developers.kakao.com/docs/en/kakaologin/common).
- [Naver는 Token Revocation API에서 access token 또는 refresh token 폐기 요청 시 연결된 token 쌍까지 함께 폐기한다고 설명한다](https://developers.naver.com/docs/login/devguide/devguide.md).
- [Google OAuth 문서는 access token 수명이 제한되어 있고, 장기 접근이 필요하면 refresh token을 안전하게 보관해야 한다고 설명한다](https://developers.google.com/identity/protocols/oauth2).
- [Microsoft Entra CAE는 token 만료시간만 기다리지 않고 critical event와 정책 평가로 접근을 재평가하는 모델을 설명한다](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation).
- [Auth0는 refresh token rotation, reuse detection, token family revocation, leeway를 설명한다](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation).
- [Netflix는 edge에서 token/identity 처리를 중앙화해 downstream 서비스의 token 처리 부담을 줄인 사례를 설명한다](https://netflixtechblog.com/edge-authentication-and-token-agnostic-identity-propagation-514e47e0b602).

## 세션/토큰 역할 분리

| 항목 | 역할 | 저장/검증 위치 | 폐기 기준 |
| --- | --- | --- | --- |
| access token | API 요청 인증과 인가 판단에 필요한 짧은 수명 증명 | client 보관, gateway/auth middleware 검증 | 만료, session 폐기, key rotation, 위험 이벤트 |
| refresh token | access token 재발급용 장기 증명 | client 안전 저장, 서버 hash 저장 | logout, 재사용 탐지, family 폐기, device 분실 |
| session | 특정 device/browser 로그인 상태 | 서버 DB 또는 session store | logout, 계정 잠금, 비밀번호 변경, 위험 이벤트 |
| device session | 사용자 기기 단위 상태와 신뢰도 | 서버 DB | 기기 로그아웃, 장기 미사용, 의심 기기 |
| auth challenge | 민감 작업의 추가 인증 결과 | 서버 DB | 만료, 성공 후 짧은 재사용 기간 종료 |

## DropMong 권장 모델

### `sessions`

| 필드 | 설명 |
| --- | --- |
| `id` | session id |
| `member_id` | 내부 회원 id |
| `device_id` | 브라우저/앱 기기 식별자 |
| `status` | active, revoked, expired |
| `auth_time` | 최초 인증 시각 |
| `last_seen_at` | 마지막 사용 시각 |
| `assurance_level` | password/social/mfa 등 현재 인증 강도 |
| `risk_level` | low, medium, high |
| `revoked_at`, `revoked_reason` | 폐기 시각과 이유 |

### `refresh_token_families`

| 필드 | 설명 |
| --- | --- |
| `id` | refresh token family id |
| `session_id` | 연결된 session |
| `member_id` | 내부 회원 id |
| `status` | active, revoked, reuse_detected |
| `created_at`, `revoked_at` | 생성/폐기 시각 |
| `reuse_detected_at` | 이미 폐기된 token 재사용 탐지 시각 |

### `refresh_tokens`

| 필드 | 설명 |
| --- | --- |
| `id` | token row id |
| `family_id` | token family |
| `token_hash` | 원문 token이 아닌 hash |
| `previous_token_id` | rotation 이전 token |
| `issued_at`, `expires_at` | 발급/만료 시각 |
| `rotated_at`, `revoked_at` | 교체/폐기 시각 |
| `client_ip`, `user_agent_hash` | 이상 징후 분석용 최소 정보 |

## API 적용안

| API | 책임 |
| --- | --- |
| `POST /auth/login` | credential 또는 social callback 검증 후 session과 token 발급 |
| `POST /auth/token/refresh` | refresh token rotation, 이전 token 폐기, reuse 탐지 |
| `POST /auth/logout` | 현재 session과 refresh token family 폐기 |
| `POST /auth/logout-all` | 회원의 모든 active session 폐기 |
| `GET /auth/sessions` | 사용자의 device session 목록 조회 |
| `DELETE /auth/sessions/{session_id}` | 특정 기기 로그아웃 |

## 확인 필요

- DropMong 초기 버전에서 access token을 JWT로 둘지, opaque token으로 두고 introspection을 할지는 성능/폐기 요구사항에 따라 결정해야 한다.
- mobile app이 없다면 refresh token 저장 방식은 웹 cookie 중심으로 먼저 설계할 수 있다.

## 출처

| 회사 | 출처 | 저장 위치 | 확인한 내용 |
| --- | --- | --- | --- |
| Kakao | https://developers.kakao.com/docs/en/kakaologin/common | companies/domestic/kakao/posts/undated-kakao-login-oauth-oidc.md | access/refresh token 만료와 refresh token 교체 |
| Naver | https://developers.naver.com/docs/login/devguide/devguide.md | companies/domestic/naver/posts/undated-naver-login-token-revocation.md | access/refresh token cascade revocation |
| Google | https://developers.google.com/identity/protocols/oauth2 | companies/global/google/posts/undated-google-openid-connect-iap.md | refresh token 장기 보관과 발급 제한 |
| Microsoft | https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation | companies/global/microsoft/posts/undated-microsoft-entra-cae-risk-mfa.md | token lifetime과 event-driven 재평가 |
| Auth0 | https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation | companies/global/auth0/posts/undated-auth0-token-rotation-account-linking-mfa.md | refresh token rotation과 reuse detection |
