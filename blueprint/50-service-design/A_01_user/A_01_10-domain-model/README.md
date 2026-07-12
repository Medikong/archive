---
id: SD.A.0110
title: Context 사용자 도메인 모델 설계 인덱스
type: service-design-domain-model-index
status: draft
tags: [service-design, user, domain-model, index]
source: local
created: 2026-07-10
updated: 2026-07-10
service_design: SD.A.01
bounded_context: BC.A.01
---

# Context 사용자 도메인 모델 설계 인덱스

## 역할

Context 사용자의 사용자 계정, 가입 연동 Process Manager, 가입 프로필 초안, 사용자 프로필과 공통 이벤트 계약을 책임별 문서로 안내한다.

## 원천

- [BC.A.01 Context 사용자 후보](../../../40-event-storming-bounded-context/BC_A_01_limited_drop_commerce.md)
- [REQ.A.05 인증·회원](../../../00-requirements/REQ_A_05_auth_member.md)
- [SD.A.300 Context 인증](../../A_300_auth/README.md)
- [SD.A.01 서비스 상세 설계](../README.md)

## 하위 문서

| 문서 | 책임 | 주요 모델 | 상태 |
| --- | --- | --- | --- |
| [가입과 계정](registration-account.md) | 가입 초안, `user_id` 발급, Auth 연동, 계정 상태 | `UserRegistrationDraft`, `UserProvisioning`, `UserAccount` | draft |
| [프로필](profile.md) | 이름·닉네임·인사말·프로필 이미지 자산 참조 | `UserProfile` | draft |
| [공통 계약](shared-contracts.md) | ID/VO, Integration Event, Rule, 외부 경계와 Hotspot | 전체 | draft |

## 결정 경계

- `UserAccount`와 `UserProfile`은 별도 Aggregate다. 가입 장기 작업은 `UserProvisioning`이 Event와 Policy로 조정한다.
- `UserRegistrationDraft`는 Auth 가입 전에 이름·닉네임 후보와 추천인 귀속 참조를 보관하고 `profile_request_id`를 발급한다. 동의 원문은 저장하지 않는다.
- Auth 연동 성공 전 `UserAccount`는 `provisioning`이며 일반 조회에서 숨긴다.
- `user_id`는 생성 뒤 변경·재사용·병합하지 않는다.
- `restriction_version`은 Auth projection의 순서를 보장하도록 단조 증가한다.
- 마이 화면의 등급·주문·쿠폰·포인트·찜·알림은 Aggregate 필드가 아니다.
