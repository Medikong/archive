---
id: SD.A.0120.READ
title: Context 사용자 조회 모델과 인덱스
type: service-design-persistence
status: draft
tags: [service-design, user, read-model, index]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 조회 모델과 인덱스

## 조회 모델

| 모델 | 포함 | 제외 |
| --- | --- | --- |
| `OwnProfile` | user ID, 상태, 프로필, media asset ID, user version | 이메일·휴대폰·role/permission |
| `UserAccountStatus` | user ID, 상태, user version, 변경 시각 | 프로필 원문과 운영 상세 사유 |

본인 프로필과 운영 상태 조회는 `users`에서 직접 읽는다. 규모나 성능 문제가 확인되기 전에는 projection을 추가하지 않는다.

## 인덱스

| 인덱스 | 용도 |
| --- | --- |
| `UNIQUE users(registration_id)` | 가입 재시도에서 같은 User 반환 |
| `users(account_status, updated_at)` | 운영 상태 조회 |
| `user_status_history(user_id, changed_at DESC)` | 사용자별 감사 이력 |

닉네임 유일성 요구가 없으므로 unique nickname 인덱스를 만들지 않는다.

## 화면 컴포넌트 조회

User는 `profile_media_asset_id`만 반환한다. 프로필 이미지 컴포넌트가 Ingress를 통해 Media에 signed URL을 요청하고, 다른 컴포넌트는 각각 소유 서비스를 호출한다. Media 실패는 User 조회 실패로 바꾸지 않는다.
