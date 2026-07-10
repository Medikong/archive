---
id: UI.A.300
group_id: UI.A.300
ui_ids: [UI.A.300, UI.A.301, UI.A.302, UI.A.303]
title: 인증 및 회원 UI
type: ui-asset
status: draft
tags: [ui, screenshot, auth, member, signin, signup, phone-signin, component-sheet, dropmong]
source: local
created: 2026-07-07
updated: 2026-07-10
---

# 인증 및 회원 UI

## 기본 정보

- UI ID: `UI.A.300`
- 포함 UI ID: `UI.A.300`, `UI.A.301`, `UI.A.302`, `UI.A.303`
- 연관 Page: [PAGE.A.300 인증 및 회원 페이지](../../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md)
- 에셋 유형: 화면 이미지, 컴포넌트 시트
- 파일 경로:
  - [로그인 메인](../assets/UI_A_300_multi_signin/UI_A_300_01_multi_signin.png)
  - [로그인 메인 컴포넌트 시트](../assets/UI_A_300_multi_signin/UI_A_300_02_multi_signin_component.png)
  - [이메일 회원가입](../assets/UI_A_301_email_signup/UI_A_301_01_email_signup.png)
  - [이메일 회원가입 컴포넌트 시트](../assets/UI_A_301_email_signup/UI_A_301_02_email_signup_component.png)
  - [이메일 로그인](../assets/UI_A_302_email_signin/UI_A_302_01_email_signin.png)
  - [이메일 로그인 컴포넌트 시트](../assets/UI_A_302_email_signin/UI_A_302_02_email_signin_component.png)
  - [휴대폰 번호 로그인](../assets/UI_A_303_phone_signin/PAGE_A_303_phone_signin.png)
- 원본 URL: local
- 캡처 일시: 2026-07-07, 2026-07-08
- 캡처 조건: DropMong 인증/회원 화면, 휴대폰/이메일 로그인, 이메일 회원가입, 가상 SMS 휴대폰 인증, 계정 복구 진입

## 연관 태그

- 요구사항 참조: [REQ.A.05](../../00-requirements/REQ_A_05_auth_member.md), [REQ.A.01](../../00-requirements/REQ_A_01_limited_drop_commerce.md), [REQ.A.02](../../00-requirements/REQ_A_02_coupon_benefit.md)
- 페이지 참조: [PAGE.A.300](../../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md)
- UC 참조: [UC.A.300](../../30-uc/UC_A_300_auth_member.md)
- 도메인 참조: [SD.A.30010](../../50-service-design/A_300_auth/A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- 영속성 참조: [SD.A.30020](../../50-service-design/A_300_auth/A_300_20-persistence/README.md)
- 서비스 참조: [SD.A.30030](../../50-service-design/A_300_auth/A_300_30-service/README.md)
- API 참조: [SD.A.30040](../../50-service-design/A_300_auth/A_300_40-api/README.md)
- 시나리오 참조: SCN.A.300 예정

## 화면 미리보기

<div style="display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 12px; width: 100%; align-items: start; padding: 8px 0;">
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_300_multi_signin/UI_A_300_01_multi_signin.png" alt="UI.A.300 로그인 메인" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.300<br/>로그인 메인</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_301_email_signup/UI_A_301_01_email_signup.png" alt="UI.A.301 이메일 회원가입" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.301<br/>이메일 회원가입</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_302_email_signin/UI_A_302_01_email_signin.png" alt="UI.A.302 이메일 로그인" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.302<br/>이메일 로그인</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_303_phone_signin/PAGE_A_303_phone_signin.png" alt="UI.A.303 휴대폰 번호 로그인" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.303<br/>휴대폰 번호 로그인</figcaption></figure>
</div>

## UI.A.300 로그인 메인

### 에셋

<img src="../assets/UI_A_300_multi_signin/UI_A_300_01_multi_signin.png" alt="로그인 메인 페이지" style="display: block; width: 50%; height: auto;" />

### 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상태 표시줄 | 모바일 상태 정보를 보여준다. | 고정 표시 |
| 2 | 브랜드 로고/워드마크 | DropMong 브랜드를 강조한다. | 브랜드 인지 |
| 3 | 히어로 헤드라인/본문 | 로그인 진입의 가치를 설명한다. | 안내 |
| 4 | 주요 CTA 버튼 | 휴대폰 번호 로그인 진입을 제공한다. | `UI.A.303` 이동 |
| 5 | 보조 로그인 버튼 | 이메일, Apple, Google 로그인을 제공한다. | 인증 방식 선택 |
| 6 | 회원가입 안내 문구 | 이메일 회원가입으로 이동한다. | `UI.A.301` 이동 |
| 7 | 신뢰/안전 정보 카드 | 안전 거래 안내를 제공한다. | 안전 안내 |
| 8 | 하단 안내/법적 문구 | 약관 동의와 고객센터 링크를 제공한다. | 약관/도움말 |

### 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 인증 | `authEntry.redirectTarget` | string? | 로그인 성공 후 복귀 화면 |
| 인증 | `authMethods.phone.enabled` | boolean | 휴대폰 로그인 버튼 표시 |
| 인증 | `authMethods.email.enabled` | boolean | 이메일 로그인 버튼 표시 |
| 인증 | `authMethods.apple.enabled` | boolean | Apple 로그인 버튼 표시 |
| 인증 | `authMethods.google.enabled` | boolean | Google 로그인 버튼 표시 |

## UI.A.301 이메일 회원가입

### 에셋

<img src="../assets/UI_A_301_email_signup/UI_A_301_01_email_signup.png" alt="이메일 회원가입 페이지" style="display: block; width: 50%; height: auto;" />

### 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상태바/앱 바 | 뒤로가기와 페이지 제목을 제공한다. | 뒤로가기 |
| 2 | 상단 프로모션 배너 | 회원가입 혜택 메시지를 전달한다. | 안내 |
| 3 | 텍스트 입력 필드 | 이름, 이메일, 비밀번호, 비밀번호 확인을 입력받는다. | 기본, 포커스, 입력 완료, 오류 |
| 4 | 휴대폰 번호 입력 필드 | 국가번호와 휴대폰 번호를 입력받는다. | 가상 SMS 인증 연결 |
| 5 | 추천인 코드 입력 필드 | 선택 추천인 코드를 입력받는다. | 선택 입력 |
| 6 | 이용약관 체크리스트 | 필수/선택 약관 동의를 받는다. | 체크/미체크 |
| 7 | 주요 CTA 버튼 | 회원가입 완료를 실행한다. | 이메일 인증과 휴대폰 인증 완료 필요 |
| 8 | 하단 로그인 안내 문구 | 이메일 로그인으로 이동한다. | `UI.A.302` 이동 |

### 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 입력 | `form.name` | string | 이름 입력 |
| 입력 | `form.email` | string | 이메일 주소 입력 |
| 입력 | `form.password` | string | 비밀번호 입력 |
| 입력 | `form.phoneNumber` | string | 휴대폰 번호 입력 |
| 검증 | `validation.email.valid` | boolean | 이메일 형식 상태 |
| 검증 | `validation.emailVerification.completed` | boolean | 이메일 인증 메일 검증 완료 여부 |
| 검증 | `validation.phoneVerification.completed` | boolean | 가상 SMS 휴대폰 인증 완료 여부 |
| 가입 | `registration.id` | string? | 서버가 발급한 회원가입 작업 식별자 |
| 가입 | `registration.expiresAt` | datetime? | 회원가입 작업 만료 시각 |
| 동의 | `agreements[].checked` | boolean | 약관 체크 상태 |
| 액션 | `actions.canSubmit` | boolean | 회원가입 완료 CTA 활성 |

## UI.A.302 이메일 로그인

### 에셋

<img src="../assets/UI_A_302_email_signin/UI_A_302_01_email_signin.png" alt="이메일 로그인 페이지" style="display: block; width: 50%; height: auto;" />

### 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상태 표시줄/앱 바 | 뒤로가기와 페이지 제목을 제공한다. | 뒤로가기 |
| 2 | 브랜드 로고/히어로 영역 | 브랜드와 마스코트 이미지를 보여준다. | 안내 |
| 3 | 이메일 입력 필드 | 이메일 주소를 입력받는다. | 기본, 포커스, 입력 완료, 오류 |
| 4 | 비밀번호 입력 필드 | 비밀번호를 입력받고 보기/숨기기를 제공한다. | 보기/숨기기 |
| 5 | 로그인 상태 유지/비밀번호 재설정 | 세션 유지와 비밀번호 복구로 이동한다. | 체크, `UI.A.310` 이동 |
| 6 | 주요 CTA 버튼 | 이메일 로그인을 실행한다. | 활성/비활성 |
| 7 | 보조 링크 영역 | 이메일 찾기, 비밀번호 재설정, 회원가입을 제공한다. | 계정 복구/가입 이동 |
| 8 | 소셜 로그인 버튼 | Apple, Google 로그인을 제공한다. | MVP 이후 후보 |

### 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 입력 | `form.email` | string | 이메일 주소 입력 |
| 입력 | `form.password` | string | 비밀번호 입력 |
| 입력 | `form.rememberMe` | boolean | 로그인 상태 유지 |
| 인증 | `auth.redirectTarget` | string? | 로그인 성공 후 복귀 화면 |
| 오류 | `error.code` | string? | 로그인 실패/계정 잠금 사유 코드 |
| 액션 | `actions.canSubmit` | boolean | 로그인 CTA 활성 |

## UI.A.303 휴대폰 번호 로그인

### 에셋

<img src="../assets/UI_A_303_phone_signin/PAGE_A_303_phone_signin.png" alt="휴대폰 번호 로그인 페이지" style="display: block; width: 50%; height: auto;" />

### 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상단 앱 바 | 로그인 메인으로 돌아갈 수 있는 헤더를 제공한다. | 뒤로가기 |
| 2 | 안내 문구 | 연결된 휴대폰 번호로 로그인한다는 맥락을 안내한다. | 안내 확인 |
| 3 | 휴대폰 번호 입력 필드 | 국가번호와 휴대폰 번호를 입력받는다. | 형식 검증 |
| 4 | 인증번호 요청/재전송 | 가상 SMS 인증번호를 요청하거나 다시 보낸다. | 활성, 쿨다운, 제한 |
| 5 | 인증번호 입력 필드 | 가상 SMS 인증번호를 입력받는다. | 기본, 포커스, 입력 완료, 오류 |
| 6 | 로그인 CTA | 인증번호 검증 후 기존 `user_id`로 로그인한다. | 활성, 비활성, 로딩 |
| 7 | 이메일 회원가입 안내 | 연결된 `user_id`가 없을 때 가입으로 이동한다. | `UI.A.301` 이동 |

### 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 휴대폰 로그인 | `phoneSignin.countryCode` | string | 국가번호 표시 |
| 휴대폰 로그인 | `phoneSignin.phoneNumber` | string | 휴대폰 번호 입력 |
| 휴대폰 로그인 | `phoneSignin.verificationCode` | string | 가상 SMS 인증번호 입력 |
| 휴대폰 로그인 | `phoneSignin.resendAvailableAt` | datetime? | 재전송 가능 시각 |
| 휴대폰 로그인 | `phoneSignin.linkedUserIdExists` | boolean? | 연결된 사용자 계정 존재 여부 |
| 인증 | `auth.redirectTarget` | string? | 로그인 성공 후 복귀 화면 |

## 설계 반영 사항

- Read Model 후보: `RM.A.300 AuthMemberReadModel`
- Command 후보: `CMD.A.40.StartPhoneSignin`, `CMD.A.41.StartEmailSignin`, `CMD.A.42.StartSocialSignin`, `CMD.A.43.GoEmailSignup`, `CMD.A.44.SubmitEmailSignup`, `CMD.A.47.SubmitEmailSignin`, `CMD.A.51.VerifyPhoneSignin`
- Error 후보: `ERR.A.40.AUTH_METHOD_UNAVAILABLE`, `ERR.A.41.AUTH_REDIRECT_INVALID`, `ERR.A.42.EMAIL_ALREADY_EXISTS`, `ERR.A.47.ACCOUNT_LOCKED`, `ERR.A.51.PHONE_AUTH_ACCOUNT_NOT_LINKED`, `ERR.A.52.PHONE_CODE_INVALID`
- 권한 후보: 비회원 접근 가능, 인증 성공 후 redirect target 또는 홈으로 복귀

## 확인 필요

- 휴대폰 번호 로그인 인증번호 TTL, 재전송 제한, 실패 횟수 제한 상태의 시각 표현을 정한다.
- 이메일 회원가입 중 휴대폰 인증 완료 상태를 같은 화면에서 처리할지 `UI.A.303`을 재사용할지 정한다.
- Apple/Google 버튼의 MVP 비활성 상태와 후속 인증 수단 연동 안내 문구를 정한다.
- 로그인 성공 후 redirect target이 없을 때 홈으로 이동하는 전환 문구를 정한다.
