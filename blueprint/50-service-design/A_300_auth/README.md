---
id: SD.A.300
title: Context 인증 서비스 상세 설계
type: service-design
status: draft
tags: [service-design, auth, identity, session]
source: local
created: 2026-07-09
updated: 2026-07-10
bounded_context: BC.A.300
---

# Context 인증 서비스 상세 설계

## 기본 정보

- Service Design ID: `SD.A.300`
- Context: Context 인증
- 상태: draft
- 책임: 인증 식별자와 credential, 소유 확인, 가입 검증 증거와 동기 회원가입 완료, 비밀번호 재설정, IdentityLink, Session, role/permission grant, 인증 정책, 감사 이벤트 전송을 구현 관점에서 상세화한다.

## 기준 결정

- 인증 유스케이스 식별자는 `UC.A.300`이다.
- `REQ.A.05`, `UC.A.300`, `BC.A.300`, `SD.A.300`을 현재 인증 설계의 기준 문서로 사용한다.
- Context 사용자가 `user_id`와 사용자 계정 생명주기를 소유한다. Context 인증은 Context 사용자가 보낸 멱등 계정 연동 요청으로만 `user_id`를 받고, 인증/세션/role mechanics만 소유한다. Auth가 Context 사용자에 `user_id` 발급을 동기 요청하지 않는다.
- 기존 `features/auth-user` 문서와 책임이 충돌하면 이 설계를 우선하고, 구현 이력 문서는 점진적으로 정리한다.
- 웹은 opaque session cookie, 모바일은 access JWT와 회전형 opaque refresh token을 기본으로 한다. 이메일·휴대폰·프로필은 token이나 내부 사용자 헤더에 넣지 않는다.

## 연관 태그

- BC 참조: [BC.A.300](../../40-event-storming-bounded-context/BC_A_300_auth_member.md), [BC.A.01](../../40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md)
- 요구사항 참조: [REQ.A.05](../../00-requirements/REQ_A_05_auth_member.md)
- UC 참조: [UC.A.300](../../30-uc/UC_A_300_auth_member.md)
- 페이지/UI 참조: [PAGE.A.300](../../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md), [PAGE.A.310](../../10-sitemap/PAGE_A_310_password_find/PAGE_A_310_password_find.md), [UI.A.300](../../20-ui/UI_A_300_auth_member/UI_A_300_auth_member.md), [UI.A.310](../../20-ui/UI_A_310_password_find/UI_A_310_password_find.md)
- 처리 시퀀스 참조: [SCN.A.300](../../80-sequence/A_300_auth/README.md)

## 설계 영역

| 영역 | 식별자 | 폴더 | 상태 |
| --- | --- | --- | --- |
| 도메인 모델 | `SD.A.30010` | [A_300_10-domain-model](A_300_10-domain-model/SD_A_30010_auth_domain_model.md) | draft |
| 영속성 | `SD.A.30020` | [A_300_20-persistence](A_300_20-persistence/README.md) | draft |
| 서비스 | `SD.A.30030` | [A_300_30-service](A_300_30-service/README.md) | draft |
| API | `SD.A.30040` | [A_300_40-api](A_300_40-api/README.md) | draft |
| 처리 시퀀스 | `SCN.A.300` | [80-sequence/A_300_auth](../../80-sequence/A_300_auth/README.md) | draft |

## 책임 경계

| Context 인증이 소유 | 다른 Context 소유와 연동 계약 |
| --- | --- |
| Identity, PasswordCredential, VerificationChallenge | `user_id` 발급과 사용자 계정 생명주기: Context 사용자. Auth의 가입 인증 완료 이벤트를 받은 뒤 Auth에 계정 연동을 요청한다. |
| IdentityLink와 인증 식별자의 영구 단일 귀속 | 이름/주소/마케팅 속성/프로필: Context 사용자 |
| Registration과 PasswordReset의 인증 단계 | 약관 원문/동의 증빙, 추천인 보상: 담당 Context |
| AuthenticationIntent, Session, SessionCredential | 구매 가능 여부와 도메인 리소스 소유권: 각 업무 Context |
| AccessGrant, UserAuthState와 최소 role/permission claim | 감사 이벤트 수집·검색·보존: Context 감사 |

## 현재 확인 필요

- Context 사용자 연동 이벤트 토픽, broker ACL, Inbox 보존 기간과 임시 사용자 계정 실패 보상 운영값.
- 이메일/SMS Challenge의 목적별 TTL, 재전송, 최대 실패 횟수.
- role/permission 원천과 grant 축소 시 Session 무효화 지연 허용치.
- 수동 인증 식별자 변경과 grant 축소 후 Session 폐기 범위. 비밀번호 재설정은 전체 폐기, 휴대폰 셀프 교체의 현재 Session은 email Link 재바인딩 후 유지로 확정했다.
- 웹 Session CSRF 도출 키 회전 주기와 모바일 기기별 refresh family 제한.
