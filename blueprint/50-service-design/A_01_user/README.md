---
id: SD.A.01
title: Context 사용자 서비스 상세 설계
type: service-design
status: draft
tags: [service-design, user, account, profile, registration, my]
source: local
created: 2026-07-10
updated: 2026-07-10
bounded_context: BC.A.01
---

# Context 사용자 서비스 상세 설계

## 기본 정보

- Service Design ID: `SD.A.01`
- Context: Context 사용자
- 물리 서비스 후보: `user-service`
- 상태: draft
- 책임: `user_id` 발급, 사용자 계정 생명주기, 가입 프로필 초안, 회원가입 비동기 연동, 사용자 표시 프로필, 계정 상태 조회를 구현 관점에서 상세화한다.

## ID 결정

- `BC.A.01`은 전체 드롭 커머스를 다루지만, 이 폴더는 그 안에서 식별한 `Context 사용자`만 설계한다.
- `BC.A.01`의 후속 설계 메모가 `SD.A.0110`, `SD.A.0120`, `SD.A.0130`, `SD.A.0140`을 지정하므로 루트 ID는 `SD.A.01`, 폴더는 `A_01_user`를 사용한다.
- 향후 주문·결제 등 다른 `BC.A.01` 후보를 상세 설계할 때는 `SD.A.01`을 재사용하지 않고 별도 업무 번호를 먼저 배정한다.

## 기준 결정

- Context 사용자가 `user_id`를 발급한다. Context 인증이 발급하거나 전달한 식별자로 사용자를 지연 생성하지 않는다.
- 회원가입은 동기 Auth 호출이 아니라 `Auth.RegistrationVerificationCompleted` version 1을 소비하고 `User.AuthLinkRequested` version 1을 발행하는 비동기 계약으로 처리한다.
- `Auth.RegistrationUserLinked` version 1을 받기 전 계정은 `provisioning`이며 일반 사용자 계정으로 노출하지 않는다.
- 이름·닉네임·프로필 이미지 자산 참조·인사말은 Context 사용자가 소유한다. 이메일·휴대폰·비밀번호·Session·role/permission grant는 Context 인증에 남긴다.
- 마이 화면 전체 조합은 BFF 책임으로 둔다. 사용자 서비스는 계정·프로필 조각만 제공하며 주문·쿠폰·포인트·등급·찜·알림 원장을 복제하지 않는다.
- 회원 등급, 배송지, 동의 원문·버전, 추천인 보상, 탈퇴 보존 정책은 아직 다른 Context와의 소유권 합의가 필요하므로 사용자 원장에 임의로 추가하지 않는다.
- 신규 구현은 현재 `service/main`의 Go Reference Service를 기준으로 Go/Chi, pgx/PostgreSQL, 별도 `server`·`worker`·`migrate` 프로세스를 사용한다. 삭제된 레거시 `user-service`의 lazy ensure 동작은 복원하지 않는다.

## 연관 태그

- BC 참조: [BC.A.01](../../40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md), [BC.A.300](../../40-event-storming-bounded-context/BC_A_300_auth_member.md)
- 요구사항 참조: [REQ.A.01](../../00-requirements/REQ_A_01_limited_drop_commerce.md), [REQ.A.05](../../00-requirements/REQ_A_05_auth_member.md)
- PAGE/UI 참조: [PAGE.A.10](../../10-sitemap/buyer-mobile-web/PAGE_A_10_my.md), [UI.A.10](../../20-ui/buyer-mobile-web/UI_A_10_my.md)
- UC 참조: [UC.A.300](../../30-uc/UC_A_300_auth_member.md)
- 인증 설계 참조: [SD.A.300](../A_300_auth/README.md), [가입 시퀀스](../../80-sequence/A_300_auth/SCN_A_300_01_email_registration.md)

## 설계 영역

| 영역 | 식별자 | 폴더 | 상태 |
| --- | --- | --- | --- |
| 도메인 모델 | `SD.A.0110` | [A_01_10-domain-model](A_01_10-domain-model/README.md) | draft |
| 영속성 | `SD.A.0120` | [A_01_20-persistence](A_01_20-persistence/README.md) | draft |
| 서비스 | `SD.A.0130` | [A_01_30-service](A_01_30-service/README.md) | draft |
| API | `SD.A.0140` | [A_01_40-api](A_01_40-api/README.md) | draft |

## 설계 문서 지도

| 영역 | 하위 문서 |
| --- | --- |
| 도메인 모델 | [가입과 계정](A_01_10-domain-model/registration-account.md), [프로필](A_01_10-domain-model/profile.md), [공통 계약](A_01_10-domain-model/shared-contracts.md) |
| 영속성 | [쓰기 모델](A_01_20-persistence/write-models.md), [신뢰성과 이벤트](A_01_20-persistence/reliability-and-events.md), [조회 모델과 인덱스](A_01_20-persistence/read-models-and-indexes.md) |
| 서비스 | [가입·계정 Handler](A_01_30-service/registration-account-handlers.md), [프로필 Handler](A_01_30-service/profile-handlers.md), [마이 조회 경계](A_01_30-service/my-query.md), [이벤트 처리](A_01_30-service/event-processing.md) |
| API | [API 공통 계약과 엔드포인트 인덱스](A_01_40-api/README.md) |

## 책임 경계

| 사용자 서비스가 소유 | 다른 Context가 소유 |
| --- | --- |
| `user_id`, 계정 생명주기와 `restriction_version` | 이메일·휴대폰·credential·Session·role/permission: Context 인증 |
| 가입 프로필 초안과 `profile_request_id` | 동의 원문·버전·철회: 동의 담당 Context |
| 이름, 닉네임, 인사말, 프로필 이미지 자산 참조 | 이미지 파일 검사·변환·보관: 프로필 미디어 시스템 |
| Auth 연동 Process Manager와 Inbox/Outbox | 인증 식별자와 `user_id` 연결: Context 인증 |
| 사용자 계정 상태와 변경 이력 | 주문·배송·쿠폰·포인트·등급·찜·알림 원장: 각 업무 Context |
| BFF에 제공할 사용자 계정·프로필 조각 | 마이 전체 조합, 메뉴와 프로모션 표시: BFF·화면 정책·프로모션 Context |

## 구현 기준

- 코드 위치 후보: `service/services/user-service`
- 구조: `cmd/server`, `cmd/worker`, `cmd/migrate`, `internal/app`, `internal/domain`, `internal/platform`, `internal/transport/http`
- 공통 패키지: `go-platform`, `go-audit`, `go-authz`, `go-contracts`
- 원장 저장소: 사용자 서비스 전용 PostgreSQL
- 비동기 계약: Kafka 호환 broker + transactional outbox/inbox
- 운영 엔드포인트: `/healthz`, `/readyz`, `/metrics`; server가 자동 migration을 실행하지 않는다.
- 목표 Principal 계약은 `SD.A.30040`의 `X-User-Id`, `X-User-Roles`, `X-Permission-Version`, `X-Token-Id`다. `X-User-Email`을 받거나 전달하지 않는다.
- 오류 응답은 `application/problem+json`으로 통일한다. 기존 `ErrorResponse`와 `X-Principal` 사용처는 별도 이행 어댑터로 다룬다.

## 현재 확인 필요

- 가입 프로필 초안 TTL과 `linkAcceptUntil` 이전 처리 여유 시간
- 닉네임 길이·금칙어·중복 허용·변경 주기와 프로필 이미지 규격
- `User.AccountAuthStateChanged` 소비 지연 시 Session 폐기 SLO와 운영 escalation 기준
- Auth 연동 거부·기한 만료 뒤 임시 계정과 프로필의 보존 기간
- 탈퇴 유예, 재가입, 개인정보 삭제와 주문·감사 기록 보존 정책
- 동의 담당 Context, 포인트·멤버십 Context, 배송지 Context의 최종 소유권
- BFF 마이 조합의 section별 timeout, cache TTL, `asOf`와 부분 실패 표현
