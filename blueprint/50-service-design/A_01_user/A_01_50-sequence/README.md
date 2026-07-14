---
id: SCN.A.01
title: Context 사용자 처리 시퀀스 인덱스
type: sequence-index
status: draft
tags: [sequence, scenario, user, registration, profile, my]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.01
domain_model: SD.A.0110
---

# Context 사용자 처리 시퀀스 인덱스

## 역할

프론트엔드, Istio Ingress Gateway, User, Auth와 Media가 참여하는 동기 처리 순서를 정리한다. 핵심 업무를 비동기 Event로 이어 붙이지 않는다.

## 문서 목록

| ID | 처리 | 문서 |
| --- | --- | --- |
| `SCN.A.01-01` | 회원가입 | [상세](SCN_A_01_01_user_provisioning_auth_link.md) |
| `SCN.A.01-02` | 본인 프로필 수정 | [상세](SCN_A_01_02_update_own_profile.md) |
| `SCN.A.01-03` | 프로필 이미지 업로드와 연결 | [상세](SCN_A_01_03_profile_image_upload_attach.md) |
| `SCN.A.01-04` | 사용자 계정 상태 변경과 Session 폐기 | [상세](SCN_A_01_04_change_user_status.md) |
| `SCN.A.01-05` | 마이 화면 컴포넌트 독립 조회 | [상세](SCN_A_01_05_load_my_components.md) |

## 설계 기준

- 가입 프론트엔드가 Auth 검증, User 생성과 Auth 완료 API를 차례로 호출한다.
- 외부 API 요청은 Istio Ingress Gateway에서 TLS 종료, 라우팅, 요청 빈도 제한, 외부에서 들어온 내부용 헤더 제거를 거친다.
- Ingress는 업무 단계를 조정하거나 여러 서비스 응답을 합치지 않는다.
- 필수 동의는 User 생성 트랜잭션에 저장한다.
- `User`와 `users.user_version` 하나만 사용한다.
- User 서비스는 Media upload intent와 signed URL을 중계하지 않는다.
- 계정 상태는 `active`, `restricted`, `deactivated` 하나로 표현한다.
- 운영 프론트엔드는 User가 서명한 상태 변경 proof를 Auth에 전달해 Session을 폐기한다.
- 마이 화면은 컴포넌트별로 소유 서비스를 호출하고 부분 실패를 독립적으로 표시한다.
- 구체적인 소비자가 없는 Event, Inbox, Outbox와 보상 Worker는 만들지 않는다.

## 연관 문서

- [사용자 서비스 상세 설계](../README.md)
- [통합 도메인 모델](../A_01_10-domain-model/README.md)
- [멱등성과 실패 처리](../A_01_20-persistence/reliability-and-events.md)
- [Application Service](../A_01_30-service/README.md)
- [API 설계](../A_01_40-api/README.md)
- [Auth 가입 시퀀스](../../../80-sequence/A_300_auth/SCN_A_300_01_email_registration.md)
