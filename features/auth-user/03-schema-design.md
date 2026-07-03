---
title: 인증과 사용자 스키마 설계
status: draft
created: 2026-07-03
updated: 2026-07-03
owner: TBD
scope: auth-user
---

# 인증과 사용자 스키마 설계

## 1. 목적

인증 방식이 웹, 모바일, 내부 QA 도구마다 달라도 내부 인가 판단을 같은 구조로 수행할 수 있도록 공통 스키마를 정의한다.

이 문서는 API 계약보다 내부 모델과 저장 구조를 먼저 다룬다.

## 검토 질문

1. Principal payload의 필수 필드는 무엇인가요?
2. anonymous Principal은 어떤 필드만 가져야 하나요?
3. user_id는 어떤 형식으로 발급하나요?
4. `auth_provider + provider_subject`에 대한 unique constraint를 어떻게 둘까요?
5. 이메일은 인증 계정의 보조 식별자로만 저장하고, 로그인 매칭에는 사용하지 않는 것으로 확정하나요?
6. ACL override의 subject/resource/action/effect 스키마를 어떤 형태로 둘까요?
7. ACL override 충돌 시 명시적 deny 우선 규칙을 스키마와 평가 모델에 어떻게 표현하나요?
8. 세션과 refresh token 저장 구조는 웹/모바일을 분리하나요?
9. Principal payload 버전을 저장하거나 전달해야 하나요?
10. 고위험 서비스용 인증 계약 v2 전환을 위해 어떤 version 필드가 필요한가요?

## 2. 공통 인가 모델

웹과 모바일은 인증 유지 방식이 다를 수 있다. 웹은 서버 세션과 보안 쿠키를 사용할 수 있고, 모바일은 access token과 refresh token을 사용할 수 있다. 하지만 업무 서비스의 인가 판단은 클라이언트별 인증 방식에 의존하지 않고 공통 모델을 사용한다.

```text
Principal
- type: anonymous | user | service
- user_id
- roles
- auth_methods
- auth_level
- session_id
- client_type
- device_id

Resource
- resource_type
- resource_id
- owner_user_id
- seller_id
- visibility
- status

ACL
- subject
- resource
- action
- effect: allow | deny
- expires_at
```

인가 판단은 Principal, Resource, RBAC policy, ACL override를 함께 사용한다. 인증 방식이 달라도 내부 서비스는 정규화된 Principal만 신뢰한다.

## 3. Principal 스키마

## 4. Resource 스키마

## 5. ACL Override 스키마

## 6. 인증 계정 연결 스키마

인증 정보와 사용자 계정 연결 정보는 테이블 관점에서 분리한다.

```text
auth_accounts
- auth_account_id
- created_at
- updated_at

auth_credentials
- credential_id
- auth_account_id
- email
- password_hash
- created_at
- updated_at

auth_provider_links
- provider_link_id
- auth_account_id
- auth_provider
- provider_subject
- provider_email
- provider_email_verified
- created_at
- updated_at

auth_user_links
- auth_user_link_id
- auth_account_id
- user_id
- created_at
- updated_at

users
- user_id
- real_name
- nickname
- profile_icon
- status
- created_at
- updated_at
```

분리 기준:

- `auth_credentials`와 `auth_provider_links`는 인증 수단과 외부 provider 식별자를 저장한다.
- `auth_user_links`는 인증 계정이 어떤 사용자 계정에 귀속되는지 저장한다.
- `users`는 사용자 도메인의 최소 사용자 정보를 저장한다.
- 로그인 매칭은 이메일이 아니라 `auth_provider + provider_subject` 또는 credential 식별 기준을 사용한다.
- 사용자 기능은 인증 수단 상세를 직접 참조하지 않고 `user_id`를 기준으로 동작한다.

## 7. 세션과 토큰 스키마

## 8. 저장소 전환 기준
