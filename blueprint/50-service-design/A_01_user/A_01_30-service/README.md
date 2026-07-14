---
id: SD.A.0130
title: Context 사용자 Application Service
type: service-design-service
status: draft
tags: [service-design, user, application-service]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 Application Service

## 역할

Application Service는 Ingress를 거쳐 들어온 Command와 Query의 사용자 Principal·권한·업무 규칙을 검증하고 `User` Aggregate를 한 번의 로컬 트랜잭션으로 변경한다. Ingress는 요청 전달만 담당한다.

## 문서

- [가입과 계정 Handler](registration-account-handlers.md)
- [프로필 Handler](profile-handlers.md)
- [마이 화면의 본인 프로필 조회](my-query.md)
- [처리 시퀀스](../A_01_50-sequence/README.md)

## Handler

| 요청 | Handler |
| --- | --- |
| `CMD.A.01-17 CreateUser` | `CreateUserHandler` |
| `CMD.A.01-20 UpdateOwnProfile` | `UpdateOwnProfileHandler` |
| `CMD.A.01-21 UpdateOwnProfileImage` | `UpdateOwnProfileImageHandler` |
| `CMD.A.01-22 ChangeUserAccountStatus` | `ChangeUserAccountStatusHandler` |
| `QRY.A.01-01 GetOwnProfile` | `GetOwnProfileHandler` |
| `QRY.A.01-03 GetUserAccountStatus` | `GetUserAccountStatusHandler` |

## 경계

- 가입 완료에서는 프론트엔드가 User와 Auth를 차례로 호출한다.
- 계정 제한에서는 운영 프론트엔드가 User와 Auth를 차례로 호출한다.
- User 서비스는 Auth·Agreement·Approval을 호출하지 않는다.
- Media 업로드와 signed URL은 프론트엔드가 Ingress를 통해 Media에 요청한다.
- 마이 화면은 기존 본인 프로필 Query를 재사용하며 전용 통합 Query를 만들지 않는다.
- 비동기 Event가 User 변경을 완료하거나 보상하지 않는다.
