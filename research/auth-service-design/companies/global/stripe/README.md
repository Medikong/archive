# Stripe

Stripe 자료는 사용자 로그인보다는 API key 운영 보안 관점이다. [restricted API key](https://docs.stripe.com/keys/restricted-api-keys), [IP restriction과 key expiry](https://docs.stripe.com/keys)를 DropMong 내부/외부 API 인증 설계 참고 자료로 본다.

## 확인한 공개 자료

| 제목 | URL | 저장 위치 |
| --- | --- | --- |
| API keys | https://docs.stripe.com/keys | posts/undated-stripe-restricted-api-keys.md |
| Restricted API keys | https://docs.stripe.com/keys/restricted-api-keys | posts/undated-stripe-restricted-api-keys.md |

## 원문 확인 내용

- [Stripe Restricted API Keys 문서](https://docs.stripe.com/keys/restricted-api-keys)는 secret key와 restricted key를 구분한다.
- [Stripe Restricted API Keys 문서](https://docs.stripe.com/keys/restricted-api-keys)는 restricted API key가 지정한 API resource와 read/write/none permission 범위에서만 동작한다고 설명한다.
- [Stripe Restricted API Keys 문서](https://docs.stripe.com/keys/restricted-api-keys)는 key가 유출되었을 때 피해 범위를 줄이기 위해 restricted key 사용을 권장한다.
- [Stripe API keys 문서](https://docs.stripe.com/keys)는 live mode key에 IP restriction을 적용할 수 있고, secret/restricted key를 expire하면 해당 key를 쓰는 code는 더 이상 API 호출을 할 수 없다고 설명한다.

## 설계 포인트

- API key는 사용자 session token보다 길게 살아남을 가능성이 높으므로 권한 축소와 위치 제한이 중요하다.
- key 관리 UI에는 생성, 권한 변경, 만료, 마지막 사용 시각, IP 제한, 소유자 표시가 필요하다.
- key 유출 대응은 "삭제"뿐 아니라 rotate, expire, audit, affected service 확인까지 포함해야 한다.

## DropMong 적용 아이디어

- 내부 service token이나 partner token은 unrestricted secret을 금지하고, 기능별 scope를 둔다.
- webhook/provider 연동 secret은 key id, hash, last_used_at, expires_at, rotated_at을 저장한다.
- 운영 도구에는 key 회전 runbook과 비상 폐기 API가 필요하다.

## 확인 필요

- [Stripe API keys 문서](https://docs.stripe.com/keys)는 API key 중심이므로 고객 인증/회원 계정 모델의 직접 근거는 아니다.
