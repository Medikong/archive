---
id: SD.A.0140
title: Context 사용자 API 설계
type: service-design-api
status: draft
tags: [service-design, user, api, rest]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 API 설계

## 공통 계약

- 외부 프론트엔드는 공개 User API를 Istio Ingress Gateway를 통해 직접 호출한다.
- Ingress는 TLS 종료, 라우팅, 요청 빈도 제한, 외부에서 들어온 내부용 헤더 제거만 담당하며 요청을 조합하지 않는다.
- 본인 API는 검증된 사용자 Principal을 요구한다.
- 운영 API는 운영자 Session, CSRF, 허용 Origin, 최근 strong auth와 canonical permission을 요구한다.
- 생성·변경 API는 `Idempotency-Key`를 요구한다. User 생성은 `registrationId`를 업무 멱등 키로 사용한다.
- 프로필·상태 변경은 `expectedUserVersion`을 요구한다.
- 이메일·휴대폰·credential·Session은 요청과 응답에 포함하지 않는다.

## API 목록

| ID | Method / Path | 역할 |
| --- | --- | --- |
| [API.A.01-01](API_A_01_01_create_user.md) | `POST /api/v1/users` | Auth 검증 완료 뒤 User 생성 |
| [API.A.01-02](API_A_01_02_get_my_profile.md) | `GET /api/v1/users/me/profile` | 본인 프로필 조회 |
| [API.A.01-03](API_A_01_03_update_my_profile.md) | `PATCH /api/v1/users/me/profile` | 본인 프로필 수정 |
| [API.A.01-05](API_A_01_05_update_my_profile_image.md) | `PUT /api/v1/users/me/profile-image` | 검증된 Media 자산 연결 |
| [API.A.01-07](API_A_01_07_change_user_account_status.md) | `POST /api/v1/operator/users/{userId}/status-transitions` | 운영 계정 상태 변경 |
| [API.A.01-08](API_A_01_08_get_user_account_status.md) | `GET /api/v1/operator/users/{userId}/status` | 운영 계정 상태 조회 |

프로필 이미지 upload intent는 User API가 제공하지 않는다. 프론트엔드가 Ingress를 통해 Media API를 직접 호출한다.

## 처리 시퀀스

| 처리 | 문서 |
| --- | --- |
| 회원가입 | [SCN.A.01-01](../../../80-sequence/A_01_user/SCN_A_01_01_user_provisioning_auth_link.md) |
| 프로필 수정 | [SCN.A.01-02](../A_01_50-sequence/SCN_A_01_02_update_own_profile.md) |
| 프로필 이미지 | [SCN.A.01-03](../A_01_50-sequence/SCN_A_01_03_profile_image_upload_attach.md) |
| 계정 상태 변경 | [SCN.A.01-04](../A_01_50-sequence/SCN_A_01_04_change_user_status.md) |
| 마이 화면 컴포넌트 조회 | [SCN.A.01-05](../A_01_50-sequence/SCN_A_01_05_load_my_components.md) |

## 공통 오류

| Code | HTTP |
| --- | --- |
| `USER_AUTHENTICATION_REQUIRED` | 401 |
| `USER_FORBIDDEN` | 403 |
| `USER_NOT_FOUND` | 404 |
| `USER_ACCOUNT_NOT_ACTIVE` | 409 |
| `USER_VERSION_CONFLICT` | 409 |
| `USER_IDEMPOTENCY_CONFLICT` | 409 |
| `USER_INPUT_INVALID` | 422 |
