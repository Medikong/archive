---
id: SD.A.0110.PROFILE
title: Context 사용자 프로필 도메인 모델
type: service-design-domain-model
status: draft
tags: [service-design, user, profile, media]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
bounded_context: BC.A.01
---

# Context 사용자 프로필 도메인 모델

## 책임

사용자 본인의 private name과 화면에 표시할 닉네임·인사말·프로필 이미지 자산 참조를 관리한다. 인증 정보와 마이 외부 요약은 포함하지 않는다.

## Aggregate

| 식별자 | 이름 | 책임 | 생명주기 |
| --- | --- | --- | --- |
| `AGG.A.01-09` | UserProfile | `user_id`별 사용자 프로필과 변경 version을 소유한다. | 기본 생성 -> 수정 -> 비식별화/보존 정책 적용 |

## 필드

| 필드 | 타입 | 공개 범위 | 규칙 |
| --- | --- | --- | --- |
| `user_id` | UserId | 내부·본인 | UserAccount와 같은 불변 식별자다. |
| `private_name` | PrivateName | 본인·인가된 업무 | 암호화 저장하고 event·로그에 넣지 않는다. |
| `nickname` | Nickname? | 표시 프로필 | NULL이면 화면 정책이 일반 기본 표시명을 사용한다. |
| `introduction` | Introduction? | 표시 프로필 | 선택 입력이며 정책 최대 길이를 넘지 않는다. |
| `profile_media_asset_id` | MediaAssetId? | 내부 참조 | 검증 완료된 미디어 자산만 연결한다. |
| `nickname_policy_version` | string? | 내부 | 현재 nickname을 승인한 정책 version이다. nickname이 없으면 NULL이다. |
| `introduction_policy_version` | string? | 내부 | 현재 introduction을 승인한 정책 version이다. introduction이 없으면 NULL이다. |
| `profile_version` | int64 | 본인·BFF | 낙관적 동시성 제어에 사용한다. |
| `created_at`, `updated_at` | Instant | 내부 | 서버 시각을 사용한다. |

## Value Object

| 이름 | 구성 | 불변조건 |
| --- | --- | --- |
| PrivateName | normalized value, encryption version | 비어 있지 않고 정책 길이를 만족한다. 원문은 로그·event·metric label 금지다. |
| Nickname | normalized value, policy version | 길이·문자·금칙어 정책을 통과한다. 중복 허용 여부는 아직 확정하지 않는다. |
| Introduction | normalized value, policy version | 제어문자와 스크립트성 입력을 제거하고 정책 길이를 만족한다. |
| MediaAssetId | opaque asset ID | 프로필 용도, 업로드 소유자, scan/transform 완료를 미디어 Port에서 확인한다. |
| ProfileVersion | positive int64 | 변경 성공마다 정확히 1 증가한다. |

## Command

| BC Command | Application Command | 규칙 |
| --- | --- | --- |
| `CMD.A.01-18 기본 프로필 생성` | CreateDefaultProfile | UserProvisioning에서만 호출하고 `user_id`별 한 번만 성공한다. |
| `CMD.A.01-20 프로필 정보 수정` | UpdateOwnProfile | Principal의 `user_id`와 대상이 같고 expected version이 일치해야 한다. |
| `CMD.A.01-21 프로필 이미지 변경` | UpdateOwnProfileImage | 검증된 미디어 자산 ID와 expected version을 요구한다. |

## Event

| Event | 공개 payload |
| --- | --- |
| `EVT.A.01-23 기본 프로필 생성됨` | `user_id`, `profile_version`, `occurred_at` |
| `EVT.A.01-25 사용자 프로필 수정됨` | `user_id`, `profile_version`, `changed_fields`, `occurred_at` |
| `EVT.A.01-26 프로필 이미지 변경됨` | `user_id`, `profile_version`, 새 asset ID의 opaque reference, `occurred_at` |

이벤트에는 private name, nickname 원문, introduction 원문, signed media URL을 넣지 않는다. 필요한 소비자는 권한 있는 사용자 API를 호출한다.

## 프로필 이미지 Policy

1. 사용자 서비스가 업로드 Intent를 요청하면 미디어 시스템이 사용자·용도에 묶인 asset ID를 발급한다.
2. 클라이언트는 미디어 시스템에 직접 업로드한다. 사용자 서비스가 binary를 proxy하지 않는다.
3. 프로필 연결 Command에서 asset의 owner, purpose=`user_profile`, scan status, transform status를 다시 확인한다.
4. 프로필 변경 트랜잭션에는 새 asset ID와 outbox만 저장한다.
5. 이전 asset 삭제는 Event 기반 비동기 정리로 요청하며 프로필 변경 성공을 되돌리지 않는다.

## Read Model

| Read Model | 포함 | 제외 |
| --- | --- | --- |
| UserProfileView | userId, private name(본인만), nickname, displayName fallback, introduction, signed profileImageUrl/expiresAt, profileVersion | 인증 식별자, 주문·쿠폰·포인트 |
| UserMySlice | userId, accountStatus, restrictionVersion, nickname, introduction, profileImageUrl/expiresAt, account/profile version | 회원 등급, 주문·배송·쿠폰·포인트·찜·알림 |

signed URL은 저장하지 않고 조회 시 미디어 Port가 발급한다. 응답에는 만료 시각을 포함하고 `Cache-Control: private, no-store`를 사용한다. 미디어 장애 때 프로필 원장은 성공으로 가장하지 않고 `profileImage.available=false`를 반환할 수 있다.

## 불변조건

- 하나의 `user_id`에는 UserProfile 하나만 존재한다.
- 계정이 `active`가 아니면 일반 본인 프로필 변경을 허용하지 않는다.
- 프로필 변경 트랜잭션은 UserAccount를 먼저 잠그고 active 상태를 다시 확인한 뒤 UserProfile을 잠근다. 계정 제한 트랜잭션이 커밋된 뒤 시작한 프로필 변경은 성공할 수 없다.
- 요청 본문의 expected version 불일치는 `409 USER_PROFILE_VERSION_CONFLICT`다.
- 요청에 없는 필드는 변경하지 않는다. NULL과 필드 생략을 구분한다.
- 닉네임 unique constraint는 중복 정책이 확정되기 전 추가하지 않는다.
- 인증 정보와 외부 업무 원장을 프로필 필드로 확장하지 않는다.

## 확인 필요

- 닉네임 중복 허용, 금칙어, 길이와 변경 횟수 제한
- introduction 최대 길이와 공개 범위
- 미디어 포맷·용량·해상도·검수·보존 기간
- 계정 탈퇴 시 private name과 프로필 자산의 삭제·익명화 시점
