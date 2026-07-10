---
id: SD.A.0130.PROFILE
title: Context 사용자 프로필 Handler 설계
type: service-design-service
status: draft
tags: [service-design, user, profile, handler, media]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
---

# Context 사용자 프로필 Handler 설계

## Handler

| 입력 | Handler | 결과 |
| --- | --- | --- |
| GetOwnProfile | GetOwnProfileHandler | 본인 프로필과 profile version, no-store 응답 |
| UpdateOwnProfile | UpdateOwnProfileHandler | 새 profile version |
| CreateProfileImageUploadIntent | CreateProfileImageUploadIntentHandler | media asset ID와 upload intent |
| UpdateOwnProfileImage | UpdateOwnProfileImageHandler | 새 asset 연결과 이전 asset 정리 Event |
| GetUserMySlice | GetUserMySliceHandler | BFF용 계정·프로필 조각 |

## 공통 인증

- Gateway/BFF가 외부 사용자 헤더를 제거하고 검증된 Principal을 생성한다.
- Principal `user_id`와 조회·변경 대상 `user_id`가 같아야 한다.
- account가 active가 아니면 일반 프로필 변경을 거부한다.
- 프로필 변경은 UserAccount `FOR SHARE` -> UserProfile `FOR UPDATE` 순서로 잠근 뒤 active를 다시 확인한다. 계정 상태 Handler의 account `FOR UPDATE`와 직렬화해 제한 트랜잭션이 커밋된 뒤 수정이 성공하는 일을 막는다.
- 이메일·휴대폰·role을 사용자 DB에서 추론하거나 프로필에 저장하지 않는다.

## UpdateOwnProfileHandler

1. actor scope에 묶인 `Idempotency-Key`, 요청 본문의 expected profile version, patch field 존재 여부를 검증한다.
2. 트랜잭션에서 UserAccount를 `FOR SHARE`, UserProfile을 `FOR UPDATE` 순서로 조회하고 active와 expected version을 다시 확인한다.
3. nickname/introduction VO 정책을 실행한다.
4. `UPDATE ... WHERE profile_version = expected`로 변경한다.
5. IdempotencyRecord와 `User.ProfileUpdated` outbox를 같은 transaction에 저장한다.
6. 새 profile version, 변경 필드와 완료 시각만 Command 결과로 반환한다. 전체 프로필과 signed URL이 필요하면 본인 프로필 Query를 다시 호출한다.

필드 누락은 유지, 명시적 NULL은 제거 요청이다. private name 변경은 일반 프로필 PATCH에 포함하지 않고 별도 고위험 정책이 확정될 때 추가한다.

## 프로필 이미지 처리

### Upload Intent

- 사용자 서비스가 미디어 Port에 user ID, purpose=`user_profile`, 콘텐츠 타입·크기 후보를 전달한다.
- 첫 로컬 트랜잭션에서 actor scope의 IdempotencyRecord를 `processing`으로 선점하고 안정적인 `media_operation_id`를 저장한다.
- Media Port의 `CreateOrGetProfileUploadIntent`에는 `media_operation_id`를 idempotency key로 전달한다. 호출은 DB 트랜잭션 밖에서 수행한다.
- Media 호출 성공 뒤 두 번째 로컬 트랜잭션에서 같은 IdempotencyRecord에 media intent/asset 참조와 만료 시각을 저장하고 `completed`로 닫는다.
- Media 호출 성공 뒤 프로세스가 죽어도 같은 operation ID로 `CreateOrGet`을 다시 호출해 기존 intent를 복구한다.
- 미디어 시스템은 opaque asset ID와 단기 upload URL을 반환한다.
- 사용자 서비스는 URL이나 binary를 저장하지 않는다.

### 자산 연결

1. actor scope의 IdempotencyRecord를 선점하고 그 record ID를 안정적인 ProfileImageChange Command ID로 사용한다.
2. asset owner와 Principal user ID가 같은지 확인한다.
3. purpose, scan status, transform status를 확인한다.
4. 변경 트랜잭션에서 UserAccount와 UserProfile을 공통 잠금 순서로 가져와 active와 expected profile version을 다시 확인한다.
5. 새 asset ID와 outbox를 UserProfile 트랜잭션에 저장한다.
6. 이전 asset은 같은 UserProfile 트랜잭션의 `User.ProfileMediaDetached` version 1 Outbox로 저장하고 Command ID를 `causation_id`로 사용한다.

Event는 `user.profile-media-detached.v1`에서 프로필 미디어 정리 worker가 소비한다. consumer는 event ID 중복 제거, asset owner/purpose와 현재 참조 상태를 확인한 뒤 보존 정책에 따라 정리한다. 미디어 정리 실패는 새 프로필 변경을 롤백하지 않고 Event를 재시도한다. 단, 새 asset 검증 실패는 변경을 허용하지 않는다.

## 프로필 정책과 미디어 Port

| Port/Policy | 입력 | 출력 |
| --- | --- | --- |
| NicknamePolicy | normalized nickname, current profile, policy version | 허용/오류 코드 |
| IntroductionPolicy | normalized introduction, policy version | 허용/오류 코드 |
| ProfileMediaPolicy | content metadata, asset scan result | 연결 허용/오류 코드 |
| MediaUploadIntentPort | user ID, purpose, content metadata, stable media operation ID | 기존 또는 신규 intent/asset ref, URL, expiresAt |

정책 운영값은 설정으로 주입하고 변경 version을 기록한다. 닉네임 유일성 정책이 없으므로 Repository 중복 조회를 불변조건으로 사용하지 않는다.

## 오류

| Code | HTTP | 조건 |
| --- | --- | --- |
| USER_PROFILE_NOT_FOUND | 404 | 활성 사용자에 프로필이 없음; 자동 생성 금지 |
| USER_PROFILE_VERSION_CONFLICT | 409 | expected version 불일치 |
| USER_PROFILE_POLICY_VIOLATION | 422 | nickname/introduction 정책 위반 |
| USER_PROFILE_MEDIA_NOT_READY | 409 | scan/transform 미완료 |
| USER_PROFILE_MEDIA_FORBIDDEN | 403 | 다른 사용자 asset |
| USER_ACCOUNT_NOT_ACTIVE | 409 | provisioning/restricted/deactivated 계정 |
| USER_IDEMPOTENCY_CONFLICT | 409 | 같은 key의 다른 payload |

## 관측성

- `user_profile_query_total{result}`
- `user_profile_update_total{result,field_group}`
- `user_profile_media_intent_total{result}`
- `user_profile_media_attach_total{result}`
- 외부 media latency histogram

nickname, private name, introduction, asset ID를 로그·metric label·trace attribute에 넣지 않는다.
