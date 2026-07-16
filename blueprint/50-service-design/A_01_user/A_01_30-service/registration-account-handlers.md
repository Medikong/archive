---
id: SD.A.0130.ACCOUNT
title: Context 사용자 가입과 계정 Handler
type: service-design-service
status: draft
tags: [service-design, user, registration, account]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 가입과 계정 Handler

## CreateUserHandler

입력은 `registrationId`, `registrationCompletionProof`, 프로필, 필수 동의 목록이다.

1. Auth 공개 키로 proof 서명, `registrationId`, purpose, audience, 필수 검증 완료, 발급·만료 시각을 확인한다.
2. 필수 동의 code와 version을 현재 설정과 비교한다.
3. `registration_id` 기존 User가 있으면 요청 fingerprint를 비교하고 같은 결과를 반환한다.
4. 신규이면 User ID를 발급한다.
5. `users`, `user_agreement_acceptances`, 생성 멱등 결과를 한 트랜잭션에 저장한다.
6. `userId`, `userVersion=1`, `createdAt`, 짧은 수명의 `userCreationProof`를 반환한다.

이 Handler는 Auth를 호출하지 않고 가입 완료 Event도 발행하지 않는다. 프론트엔드가 다음 단계에서 Auth 가입 완료 API를 호출한다.

## ChangeUserAccountStatusHandler

입력은 target user ID, `targetStatus`, `reasonCode`, `expectedUserVersion`, 운영 Principal과 `Idempotency-Key`다.

1. 운영자 Session, 최근 strong auth와 canonical permission을 확인한다.
2. `active`, `restricted`, `deactivated` 사이의 허용 전이를 확인한다.
3. `user_id`, 현재 상태와 `expectedUserVersion`을 조건으로 User를 갱신한다.
4. `status_change_id`, 상태 이력과 멱등 결과를 같은 트랜잭션에 저장한다.
5. `statusChangeId`, 새 `accountStatus`, `userVersion`과 Auth 전용 `userStatusChangeProof`를 반환한다.

운영 프론트엔드는 `userStatusChangeProof`로 Auth API를 동기 호출한다. 제한·비활성에서는 Auth가 Session을 폐기한다. Auth 호출 실패 시 프론트엔드는 같은 proof로 Auth 요청만 재시도한다.

## 오류

| Code | 조건 |
| --- | --- |
| `USER_REGISTRATION_PROOF_INVALID` | Auth proof 무효·만료·registration 불일치 |
| `USER_REQUIRED_AGREEMENT_INVALID` | 필수 동의 누락 또는 version 불일치 |
| `USER_REGISTRATION_CONFLICT` | 같은 registration에 다른 생성 요청 |
| `USER_ACCOUNT_STATUS_PERMISSION_DENIED` | 운영 권한 부족 |
| `USER_ACCOUNT_STATUS_TRANSITION_INVALID` | 허용되지 않은 상태 전이 |
| `USER_VERSION_CONFLICT` | expected version 불일치 |
| `USER_IDEMPOTENCY_CONFLICT` | 같은 key의 다른 요청 |

## 관련 시퀀스

- [회원가입과 자동 로그인](../../../80-sequence/A_01_user/SCN_A_01_01_user_provisioning_auth_link.md)
- [계정 상태 변경](../A_01_50-sequence/SCN_A_01_04_change_user_status.md)
