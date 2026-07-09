---
id: UI.A.310
group_id: UI.A.310
ui_ids: [UI.A.310, UI.A.311, UI.A.312, UI.A.313, UI.A.314]
title: 비밀번호 재설정 UI
type: ui-asset
status: draft
tags: [ui, screenshot, auth, password-reset, recovery, dropmong]
source: local
created: 2026-07-07
updated: 2026-07-07
---

# 비밀번호 재설정 UI

## 기본 정보

- UI ID: `UI.A.310`
- 포함 UI ID: `UI.A.310`, `UI.A.311`, `UI.A.312`, `UI.A.313`, `UI.A.314`
- 연관 Page: [PAGE.A.310](../../10-sitemap/PAGE_A_310_password_find/PAGE_A_310_password_find.md)
- 에셋 유형: 화면 이미지
- 파일 경로:
  - [비밀번호 찾기](../assets/UI_A_310_password_find/UI_A_310_01_password_find.png)
  - [인증 방식 선택](../assets/UI_A_310_password_find/UI_A_310_02_authorize_choose.png)
  - [휴대폰 번호 인증](../assets/UI_A_310_password_find/UI_A_310_03_phone_authorize.png)
  - [이메일 인증](../assets/UI_A_310_password_find/UI_A_310_04_email_authorize.png)
  - [새 비밀번호 설정](../assets/UI_A_310_password_find/UI_A_310_05_reset_password.png)
- 원본 URL: local
- 캡처 일시: 2026-07-07
- 캡처 조건: DropMong 비밀번호 찾기, 인증 방식 선택, 휴대폰 번호 인증, 이메일 인증, 새 비밀번호 설정 화면

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.05](../../00-requirements/REQ_A_05_auth_member.md) | 페이지 참조: [PAGE.A.310](../../10-sitemap/PAGE_A_310_password_find/PAGE_A_310_password_find.md) | UC 참조: [UC.A.300](../../30-uc/UC_A_300_auth_member.md) | 영속성 참조: PST.A.310 예정 | 서비스 참조: SVC.A.310 예정 | 시나리오 참조: SCN.A.310 예정 | API 참조: API.A.310 예정

## 화면 미리보기

<div style="display: grid; grid-template-columns: repeat(5, minmax(0, 1fr)); gap: 12px; width: 100%; align-items: start; padding: 8px 0;">
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_310_password_find/UI_A_310_01_password_find.png" alt="UI.A.310 비밀번호 찾기" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.310<br/>비밀번호 찾기</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_310_password_find/UI_A_310_02_authorize_choose.png" alt="UI.A.311 인증 방식 선택" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.311<br/>인증 방식 선택</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_310_password_find/UI_A_310_03_phone_authorize.png" alt="UI.A.312 휴대폰 번호 인증" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.312<br/>휴대폰 번호 인증</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_310_password_find/UI_A_310_04_email_authorize.png" alt="UI.A.313 이메일 인증" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.313<br/>이메일 인증</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_310_password_find/UI_A_310_05_reset_password.png" alt="UI.A.314 새 비밀번호 설정" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.314<br/>새 비밀번호 설정</figcaption></figure>
</div>

## UI.A.310 비밀번호 찾기

### 에셋

<img src="../assets/UI_A_310_password_find/UI_A_310_01_password_find.png" alt="비밀번호 찾기 페이지" style="display: block; width: 50%; height: auto;" />

### 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상단 앱 바 | 이전 화면으로 돌아갈 수 있는 진입 헤더를 제공한다. | 뒤로가기 |
| 2 | 안내 문구 | 이메일 또는 휴대폰 번호로 비밀번호 재설정을 시작할 수 있음을 안내한다. | 안내 확인 |
| 3 | 계정 식별자 입력 필드 | 이메일 주소 또는 휴대폰 번호를 입력받는다. | 기본, 포커스, 입력 완료, 오류 |
| 4 | 다음 CTA | 계정 식별자 검증 후 인증 방식 선택으로 진행한다. | 활성, 비활성, 로딩 |
| 5 | 로그인 복귀 링크 | 비밀번호 재설정을 중단하고 로그인 화면으로 돌아간다. | 링크 이동 |

### 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 비밀번호 재설정 | `passwordReset.identifier` | string | 이메일 주소 또는 휴대폰 번호 입력 |
| 비밀번호 재설정 | `passwordReset.identifierType` | enum | 이메일/휴대폰 번호 형식 구분 |
| 비밀번호 재설정 | `passwordReset.intentId` | string? | 서버 검증 후 생성되는 재설정 intent |

## UI.A.311 인증 방식 선택

### 에셋

<img src="../assets/UI_A_310_password_find/UI_A_310_02_authorize_choose.png" alt="인증 방식 선택 페이지" style="display: block; width: 50%; height: auto;" />

### 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상단 앱 바 | 이전 단계로 돌아갈 수 있는 헤더를 제공한다. | 뒤로가기 |
| 2 | 계정 확인 안내 | 재설정 대상 계정과 인증 방식 선택 맥락을 안내한다. | 안내 확인 |
| 3 | 이메일 인증 선택 카드 | 이메일 인증 경로를 선택한다. | 기본, 선택됨, 비활성 |
| 4 | 휴대폰 인증 선택 카드 | 휴대폰 번호 인증 경로를 선택한다. | 기본, 선택됨, 비활성 |
| 5 | 다음 CTA | 선택한 인증 방식의 인증 화면으로 진행한다. | 활성, 비활성 |

### 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 비밀번호 재설정 | `passwordReset.intentId` | string | 재설정 과정 식별 |
| 비밀번호 재설정 | `passwordReset.availableMethods` | enum[] | 사용 가능한 인증 수단 |
| 비밀번호 재설정 | `passwordReset.selectedMethod` | enum | 선택한 인증 방식 |

## UI.A.312 휴대폰 번호 인증

### 에셋

<img src="../assets/UI_A_310_password_find/UI_A_310_03_phone_authorize.png" alt="휴대폰 번호 인증 페이지" style="display: block; width: 50%; height: auto;" />

### 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상단 앱 바 | 인증 방식 선택 단계로 돌아갈 수 있는 헤더를 제공한다. | 뒤로가기 |
| 2 | 휴대폰 번호 안내 | 인증번호가 발송된 마스킹 휴대폰 번호를 보여준다. | 안내 확인 |
| 3 | 인증번호 입력 필드 | 가상 SMS 인증번호를 입력받는다. | 기본, 포커스, 입력 완료, 오류 |
| 4 | 재전송 버튼 | 인증번호를 다시 요청한다. | 활성, 쿨다운, 제한 |
| 5 | 인증 CTA | 인증번호 검증 후 새 비밀번호 설정으로 진행한다. | 활성, 비활성, 로딩 |

### 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 비밀번호 재설정 | `passwordReset.intentId` | string | 재설정 과정 식별 |
| 비밀번호 재설정 | `passwordReset.maskedPhoneNumber` | string | 마스킹된 휴대폰 번호 표시 |
| 비밀번호 재설정 | `passwordReset.phoneCode` | string | 인증번호 입력값 |
| 비밀번호 재설정 | `passwordReset.resendAvailableAt` | datetime? | 재전송 가능 시각 |

## UI.A.313 이메일 인증

### 에셋

<img src="../assets/UI_A_310_password_find/UI_A_310_04_email_authorize.png" alt="이메일 인증 페이지" style="display: block; width: 50%; height: auto;" />

### 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상단 앱 바 | 인증 방식 선택 단계로 돌아갈 수 있는 헤더를 제공한다. | 뒤로가기 |
| 2 | 이메일 주소 안내 | 인증 정보가 발송된 마스킹 이메일 주소를 보여준다. | 안내 확인 |
| 3 | 인증번호 입력 필드 | 이메일 인증번호를 입력받는다. | 기본, 포커스, 입력 완료, 오류 |
| 4 | 재발송 버튼 | 인증 메일을 다시 요청한다. | 활성, 쿨다운, 제한 |
| 5 | 인증 CTA | 이메일 인증 후 새 비밀번호 설정으로 진행한다. | 활성, 비활성, 로딩 |

### 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 비밀번호 재설정 | `passwordReset.intentId` | string | 재설정 과정 식별 |
| 비밀번호 재설정 | `passwordReset.maskedEmail` | string | 마스킹된 이메일 주소 표시 |
| 비밀번호 재설정 | `passwordReset.emailCode` | string | 인증번호 입력값 |
| 비밀번호 재설정 | `passwordReset.resendAvailableAt` | datetime? | 재발송 가능 시각 |

## UI.A.314 새 비밀번호 설정

### 에셋

<img src="../assets/UI_A_310_password_find/UI_A_310_05_reset_password.png" alt="새 비밀번호 설정 페이지" style="display: block; width: 50%; height: auto;" />

### 화면 구성

| 번호 | 컴포넌트 | 역할 | 주요 상태/행동 |
| --- | --- | --- | --- |
| 1 | 상단 앱 바 | 이전 인증 단계로 돌아갈 수 있는 헤더를 제공한다. | 뒤로가기 |
| 2 | 새 비밀번호 입력 필드 | 새 비밀번호를 입력받는다. | 기본, 포커스, 입력 완료, 오류, 보기/숨기기 |
| 3 | 비밀번호 확인 필드 | 새 비밀번호와 동일한 값을 확인한다. | 기본, 포커스, 입력 완료, 불일치 오류 |
| 4 | 비밀번호 규칙 안내 | 길이와 문자 조합 같은 정책 충족 여부를 보여준다. | 충족, 미충족 |
| 5 | 재설정 완료 CTA | 비밀번호 변경을 완료한다. | 활성, 비활성, 로딩 |

### 화면에 필요한 정보

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 비밀번호 재설정 | `passwordReset.intentId` | string | 재설정 과정 식별 |
| 비밀번호 재설정 | `passwordReset.newPassword` | string | 새 비밀번호 입력 |
| 비밀번호 재설정 | `passwordReset.confirmPassword` | string | 비밀번호 확인 입력 |
| 비밀번호 재설정 | `passwordReset.passwordPolicyStatus` | object | 비밀번호 규칙 충족 상태 |

## 설계 반영 사항

- Read Model 후보: `RM.A.310 PasswordResetFlowModel`
- Command 후보: `CMD.A.54.StartPasswordReset`, `CMD.A.55.SelectPasswordResetMethod`, `CMD.A.56.VerifyPasswordResetCode`, `CMD.A.57.SubmitNewPassword`
- Error 후보: `ERR.A.55.PASSWORD_RESET_IDENTIFIER_INVALID`, `ERR.A.56.PASSWORD_RESET_METHOD_UNAVAILABLE`, `ERR.A.57.PASSWORD_RESET_CODE_INVALID`, `ERR.A.58.PASSWORD_RESET_INTENT_EXPIRED`, `ERR.A.59.PASSWORD_POLICY_NOT_MET`
- 권한 후보: 비회원 접근 가능, 재설정 intent 검증 필요

## 확인 필요

- UI 문구와 실제 인증 정책 문구 일치 여부
- 인증번호 TTL, 재전송 제한, 실패 횟수 제한 상태의 시각 표현
- 이메일 인증 링크 방식과 인증번호 방식 중 MVP 기본값
- 비밀번호 재설정 완료 후 자동 로그인 여부
