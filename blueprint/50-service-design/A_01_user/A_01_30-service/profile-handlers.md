---
id: SD.A.0130.PROFILE
title: Context 사용자 프로필 Handler
type: service-design-service
status: draft
tags: [service-design, user, profile, media]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 프로필 Handler

## UpdateOwnProfileHandler

1. Principal user ID와 대상이 같은지 확인한다.
2. nickname과 introduction을 정규화하고 정책을 검증한다.
3. `user_id`, `account_status=active`, `expectedUserVersion`을 조건으로 한 UPDATE를 실행한다.
4. 멱등 결과를 같은 트랜잭션에 저장한다.
5. 새 `userVersion`, 변경 필드와 완료 시각을 반환한다.

별도 SELECT 잠금, 프로필 version과 프로필 변경 Event는 사용하지 않는다.

## UpdateOwnProfileImageHandler

프로필 이미지 업로드와 검사는 프론트엔드가 Ingress를 통해 Media에 먼저 요청한다. User 요청에는 `mediaAssetId`, Media가 발급한 `mediaAssetProof`, `expectedUserVersion`이 들어온다.

1. Media 공개 키로 proof의 owner user ID, purpose=`user_profile`, asset ID, 검사 완료와 만료 시각을 확인한다.
2. User를 조건부 UPDATE하여 `profile_media_asset_id`와 `user_version`을 바꾼다.
3. 멱등 결과를 같은 트랜잭션에 저장한다.

User 서비스는 upload URL, signed URL, binary와 Media upload intent를 저장하지 않는다. 이전 자산 정리는 Media의 미참조 자산 보존 정책에 맡기며 사용자 Event를 만들지 않는다.

## Query Handler

`GetOwnProfileHandler`는 프로필과 현재 media asset ID를 반환한다. signed URL이 필요하면 프론트엔드의 이미지 컴포넌트가 Ingress를 통해 Media에 요청한다.

## 오류

| Code | 조건 |
| --- | --- |
| `USER_ACCOUNT_NOT_ACTIVE` | 계정 상태가 active가 아님 |
| `USER_VERSION_CONFLICT` | expected version 불일치 |
| `USER_PROFILE_POLICY_VIOLATION` | 프로필 정책 위반 |
| `USER_PROFILE_MEDIA_PROOF_INVALID` | Media 증거 무효·만료·owner/purpose 불일치 |
| `USER_IDEMPOTENCY_CONFLICT` | 같은 key의 다른 요청 |

## 관련 시퀀스

- [본인 프로필 수정](../A_01_50-sequence/SCN_A_01_02_update_own_profile.md)
- [프로필 이미지 업로드와 연결](../A_01_50-sequence/SCN_A_01_03_profile_image_upload_attach.md)
