---
id: SD.A.0120.RELIABILITY
title: Context 사용자 멱등성과 실패 처리
type: service-design-persistence
status: draft
tags: [service-design, user, idempotency, transaction]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 멱등성과 실패 처리

## 원칙

- 가입은 프론트엔드의 직접 호출로 완료한다.
- 계정 제한은 운영 프론트엔드의 동기 호출로 완료한다.
- User 서비스는 자기 DB 트랜잭션만 책임진다.
- 프론트엔드는 맡은 요청을 같은 업무 키나 단기 proof로 재시도하고 서비스는 기존 결과를 반환한다.
- 가입·프로필·상태 변경을 진행시키기 위한 Event, Inbox, Outbox와 보상 Worker는 두지 않는다.

## 멱등 범위

| 작업 | scope | key |
| --- | --- | --- |
| User 생성 | `registration_id` | `registration_id` 자체 |
| 프로필 수정 | `user_id + API.A.01-03` | `Idempotency-Key` |
| 이미지 연결 | `user_id + API.A.01-05` | `Idempotency-Key` |
| 상태 변경 | `user_id + API.A.01-07` | `Idempotency-Key` |

같은 key와 같은 요청은 기존 결과를 반환하고 다른 요청은 `409 USER_IDEMPOTENCY_CONFLICT`로 거부한다.

## 가입 실패 처리

| 실패 지점 | 처리 |
| --- | --- |
| Auth 검증 증거 무효·만료 | User를 만들지 않고 `403 USER_REGISTRATION_PROOF_INVALID` |
| 필수 동의 누락·version 불일치 | User를 만들지 않고 `422 USER_REQUIRED_AGREEMENT_INVALID` |
| User 저장 실패 | 전체 롤백. 프론트엔드는 같은 `registration_id`로 재시도 |
| User 응답 유실 | 재시도 시 UNIQUE `registration_id`로 같은 `user_id` 반환 |
| Auth 가입 완료 실패 | User 결과는 유지. 프론트엔드가 User 결과를 재조회한 뒤 Auth를 다시 호출 |

Auth 가입 완료 전에는 Session과 사용자 Principal이 없으므로 생성된 User를 외부에서 사용할 수 없다. User는 Auth 연결 상태를 별도로 저장하거나 추측하지 않는다.

## 상태 변경 실패 처리

User는 상태 변경 결과를 `auth-service` 전용 단기 proof로 서명한다. 운영 프론트엔드는 Auth 호출이 실패하면 User 상태를 되돌리지 않고 같은 proof로 Auth 요청만 재시도한다. proof가 만료되면 같은 멱등 키로 User의 기존 결과와 새 proof를 받은 뒤 Auth 요청을 재시도한다. Auth는 proof의 서명·audience·만료·사용자 일치를 검증하고 같은 version을 멱등 적용한다.

## Event 정책

현재 가입·프로필·계정 상태 변경에 필수인 비동기 소비자는 없다. 따라서 Event 계약을 만들지 않는다. 향후 구체적인 소비자가 생기면 완료 사실을 알리는 Event만 추가하며, Event가 핵심 업무를 완성하게 하지 않는다.
