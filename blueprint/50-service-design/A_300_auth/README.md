---
id: SD.A.300
title: Context 인증 서비스 상세 설계
type: service-design
status: draft
tags: [service-design, auth, identity, session]
source: local
created: 2026-07-09
updated: 2026-07-15
bounded_context: BC.A.300
---

# Context 인증 서비스 상세 설계

## 기본 정보

- Service Design ID: `SD.A.300`
- Context: Context 인증
- 상태: draft
- 책임: 인증 식별자와 credential, 소유 확인, 가입 검증 증거와 동기 회원가입 완료, 비밀번호 재설정, IdentityLink, Session, access JWT, 공통 refresh rotation, 인증 정책과 감사 이벤트 전송을 구현 관점에서 상세화한다.

## 기준 결정

- 인증 유스케이스 식별자는 `UC.A.300`이다.
- `REQ.A.05`, `UC.A.300`, `BC.A.300`, `SD.A.300`을 현재 인증 설계의 기준 문서로 사용한다.
- Context 사용자가 `user_id`와 사용자 계정 생명주기를 소유한다. Context 인증은 Context 사용자가 보낸 멱등 계정 연동 요청으로만 `user_id`를 받고 인증, Session과 token 생명주기만 소유한다. Auth가 Context 사용자에 `user_id` 발급을 동기 요청하지 않는다.
- 기존 `features/auth-user` 문서와 책임이 충돌하면 이 설계를 우선하고, 구현 이력 문서는 점진적으로 정리한다.
- 웹과 모바일은 짧은 수명의 access JWT를 Bearer credential로 사용한다. 웹 refresh token은 HttpOnly·Secure·SameSite cookie로, 모바일 refresh token은 플랫폼 보안 저장소로 전달하며 Auth는 두 채널의 Session과 회전형 opaque refresh token을 서버에서 관리한다. 이메일·휴대폰·프로필은 token이나 내부 사용자 헤더에 넣지 않는다.
- JWT claim, JWKS와 Istio 검증 및 헤더 기준은 [SD.A.300.JWT](jwt-jwks-istio.md)을 단일 기준으로 사용한다.

## 연관 태그

- BC 참조: [BC.A.300](../../40-event-storming-bounded-context/BC_A_300_auth_member.md), [BC.A.01](../../40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md)
- 요구사항 참조: [REQ.A.05](../../00-requirements/REQ_A_05_auth_member.md)
- UC 참조: [UC.A.300](../../30-uc/UC_A_300_auth_member.md)
- 페이지/UI 참조: [PAGE.A.300](../../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md), [PAGE.A.310](../../10-sitemap/PAGE_A_310_password_find/PAGE_A_310_password_find.md), [UI.A.300](../../20-ui/UI_A_300_auth_member/UI_A_300_auth_member.md), [UI.A.310](../../20-ui/UI_A_310_password_find/UI_A_310_password_find.md)
- 처리 시퀀스 참조: [SCN.A.300](A_300_50-sequence/README.md)

## 설계 영역

| 영역 | 식별자 | 폴더 | 상태 |
| --- | --- | --- | --- |
| 도메인 모델 | `SD.A.30010` | [A_300_10-domain-model](A_300_10-domain-model/SD_A_30010_auth_domain_model.md) | draft |
| 영속성 | `SD.A.30020` | [A_300_20-persistence](A_300_20-persistence/README.md) | draft |
| 서비스 | `SD.A.30030` | [A_300_30-service](A_300_30-service/README.md) | draft |
| API | `SD.A.30040` | [A_300_40-api](A_300_40-api/README.md) | draft |
| 처리 시퀀스 | `SCN.A.300` | [A_300_50-sequence](A_300_50-sequence/README.md) | draft |
| JWT/JWKS/Istio 인증 처리 기준 | `SD.A.300.JWT` | [jwt-jwks-istio.md](jwt-jwks-istio.md) | draft |

## 책임 경계

| Context 인증이 소유 | 다른 Context와의 연동 기준 |
| --- | --- |
| Identity, PasswordCredential, VerificationChallenge | `user_id` 발급과 사용자 계정 생명주기: Context 사용자. 프론트엔드는 Auth proof로 User를 생성한 뒤 User proof를 Auth 완료 API에 전달한다. |
| IdentityLink와 인증 식별자의 영구 단일 귀속 | 이름/주소/마케팅 속성/프로필: Context 사용자 |
| Registration과 PasswordReset의 인증 단계 | 약관 원문/동의 증빙, 추천인 보상: 담당 Context |
| AuthenticationIntent, Session, SessionCredential | 구매 가능 여부와 도메인 리소스 소유권: 각 업무 Context |
| UserAuthState, access JWT 발급과 refresh rotation | 감사 이벤트 수집·검색·보존: Context 감사 |

## 현재 확인 필요

- User proof 검증 키의 배포·회전 방식과 회원가입 완료 재시도 보존 기간.
- 이메일/SMS Challenge의 목적별 TTL, 재전송, 최대 실패 횟수.
- 별도 인가 경계가 Auth 운영 API에 전달할 authorization decision reference 검증 방식.
- 수동 인증 식별자 변경 후 Session 폐기 범위. 비밀번호 재설정은 전체 폐기, 휴대폰 셀프 교체의 현재 Session은 email Link 재바인딩 후 유지로 확정했다.
- 웹 refresh cookie 정책과 채널별 refresh family 제한.
