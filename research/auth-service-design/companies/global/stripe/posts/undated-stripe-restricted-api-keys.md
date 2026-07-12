# Stripe Restricted API Keys

- 회사: Stripe
- 원문: https://docs.stripe.com/keys
- 원문: https://docs.stripe.com/keys/restricted-api-keys
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: API key, restricted key, IP restriction, key expiry

## 원문 확인 내용

[Stripe Restricted API Keys 문서](https://docs.stripe.com/keys/restricted-api-keys)는 restricted API key가 server-side application이 Stripe API 요청에 사용하는 제한된 key라고 설명한다. secret key와 달리 지정한 resource와 permission 범위 안에서만 동작한다.

[Stripe API keys 문서](https://docs.stripe.com/keys)는 live mode key에 IP restriction을 적용할 수 있고, secret 또는 restricted key를 expire하면 해당 key를 쓰는 code가 더 이상 API 호출을 할 수 없다고 설명한다.

## 적용 아이디어

- DropMong partner/internal API key는 모든 API 접근 권한을 주지 않는다.
- key마다 `scope`, `allowed_ip_cidr`, `expires_at`, `last_used_at`, `revoked_at`을 둔다.
- key 유출 시 affected key만 폐기할 수 있도록 key id와 hash를 분리 저장한다.

## 확인 필요

- [Stripe API keys 문서](https://docs.stripe.com/keys)는 customer account 인증 자료가 아니므로 회원 로그인 설계에는 제한적으로만 참고한다.
