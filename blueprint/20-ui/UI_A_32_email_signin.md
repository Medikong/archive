---
id: UI.A.32
title: 이메일 로그인 페이지 UI
type: ui-asset
status: draft
tags: [ui, screenshot, auth, signin, email, social-login, component-sheet, dropmong]
source: local
created: 2026-07-07
updated: 2026-07-07
---

# 이메일 로그인 페이지 UI

## 기본 정보

- UI ID: `UI.A.32`
- 연관 Page: [PAGE.A.32](../10-sitemap/PAGE_A_32_email_signin.md)
- 에셋 유형: 화면 이미지, 컴포넌트 시트
- 파일 경로:
  - [이메일 로그인 페이지](assets/UI_A_32_email_signin/UI_A_32_01_email_signin.png)
  - [이메일 로그인 페이지 컴포넌트 시트](assets/UI_A_32_email_signin/UI_A_32_02_email_signin_component.png)
- 원본 URL: local
- 캡처 일시: 2026-07-07
- 캡처 조건: DropMong 이메일 로그인, 이메일/비밀번호 입력, 로그인 상태 유지, 계정 복구 링크, 소셜 로그인, 하단 탭 상태

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.01](../00-requirements/REQ_A_01_limited_drop_commerce.md) | 페이지 참조: [PAGE.A.32](../10-sitemap/PAGE_A_32_email_signin.md) | UC 참조: UC.A.32 | 영속성 참조: PST.A.32 | 서비스 참조: SVC.A.32 | 시나리오 참조: SCN.A.32 | API 참조: API.A.32

## 에셋

### 이메일 로그인 페이지

![이메일 로그인 페이지](assets/UI_A_32_email_signin/UI_A_32_01_email_signin.png)

### 컴포넌트 시트

![이메일 로그인 페이지 컴포넌트 시트](assets/UI_A_32_email_signin/UI_A_32_02_email_signin_component.png)

## 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상태 표시줄/앱 바 | 뒤로가기와 페이지 제목을 제공한다. | 뒤로가기 |
| 2 | 브랜드 로고/히어로 영역 | 브랜드와 마스코트 이미지를 보여준다. | 안내 |
| 3 | 이메일 입력 필드 | 이메일 주소를 입력받는다. | 기본, 포커스, 입력 완료, 오류 |
| 4 | 비밀번호 입력 필드 | 비밀번호를 입력받고 보기/숨기기를 제공한다. | 보기/숨기기 |
| 5 | 로그인 상태 유지/비밀번호 재설정 | 세션 유지와 비밀번호 복구로 이동한다. | 체크, 링크 |
| 6 | 주요 CTA 버튼 | 이메일 로그인을 실행한다. | 활성/비활성 |
| 7 | 보조 링크 영역 | 이메일 찾기, 비밀번호 재설정, 회원가입을 제공한다. | 링크 이동 |
| 8 | 구분선/간편 로그인 라벨 | 이메일 로그인과 소셜 로그인을 구분한다. | 시각 구분 |
| 9 | 소셜 로그인 버튼 | Apple, Google 로그인을 제공한다. | 소셜 인증 |
| 10 | 하단 내비게이션 바 | 홈, 드롭, 알림, 장바구니, 마이 탭을 제공한다. | 전역 탭 이동 |
| 11 | 아이콘 샘플 | 뒤로가기, 이메일, 잠금, 보기, 숨기기, 체크 아이콘을 정의한다. | 아이콘 상태 |
| 12 | 폼 상태/버튼 스타일 | 입력 필드와 버튼의 기본/포커스/오류/비활성 상태를 정의한다. | 폼 상태 |

## 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 입력 | `form.email` | string | 이메일 주소 입력 |
| 입력 | `form.password` | string | 비밀번호 입력 |
| 입력 | `form.rememberMe` | boolean | 로그인 상태 유지 |
| 검증 | `validation.email.valid` | boolean | 이메일 형식 검증 |
| 검증 | `validation.password.required` | boolean | 비밀번호 필수값 검증 |
| 인증 | `auth.redirectTarget` | string? | 로그인 성공 후 복귀 화면 |
| 인증 | `auth.social.apple.enabled` | boolean | Apple 로그인 표시 |
| 인증 | `auth.social.google.enabled` | boolean | Google 로그인 표시 |
| 액션 | `actions.canSubmit` | boolean | 로그인 CTA 활성 |
| 오류 | `error.message` | string? | 로그인 실패 사유 표시 |

## 설계 반영 사항

- Read Model 후보: `RM.A.32 EmailSigninFormModel`
- Command 후보: `CMD.A.47.SubmitEmailSignin`, `CMD.A.48.ToggleRememberMe`, `CMD.A.49.StartPasswordReset`, `CMD.A.50.StartSocialSigninFromEmail`
- Error 후보: `ERR.A.46.INVALID_EMAIL_OR_PASSWORD`, `ERR.A.47.ACCOUNT_LOCKED`, `ERR.A.48.EMAIL_SIGNIN_FAILED`
- 권한 후보: 비회원 접근 가능

## 확인 필요

- 로그인 상태 유지 세션 만료 정책
- 로그인 실패 횟수 제한과 계정 잠금 정책
- 이메일 찾기/비밀번호 재설정 화면 분리 여부
- 소셜 로그인과 이메일 계정 병합 정책
- 하단 내비게이션을 인증 화면에도 유지할지 여부
