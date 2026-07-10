---
id: REQ.A.05
title: 인증 및 회원 요구사항 정의
type: requirements
status: draft
tags: [requirements, dropmong, auth, member, signup, signin, session, identity]
source: local
created: 2026-07-07
updated: 2026-07-10
---

# 인증 및 회원 요구사항 정의

## 기본 정보

- Requirements ID: `REQ.A.05`
- 프로젝트: DropMong
- 작성 기준일: 2026-07-07
- 주요 사용자: 비회원, 구매자, 판매자, 플랫폼 운영자
- 주요 이해관계자: CS 담당자, 상품/콘텐츠 검수 담당자, 인시던트 담당자, 개발/SRE 담당자, 보안/정책 담당자
- 설계 범위: 비회원 탐색, 이메일 회원가입, 이메일 회원가입 중 가상 SMS 휴대폰 인증 식별자 연동, 이메일 로그인, 연동된 휴대폰 인증 식별자 기반 로그인, 휴대폰 번호 셀프 변경, 세션/토큰 유지, 로그아웃, 인증 게이트, 인증 수단 연동, 판매자/운영자 강한 인증 요구, 인증 감사 이벤트
- 제외 범위: 결제수단 본인 인증, PG 추가 인증, 판매자 사업자 심사, 최종 정산 주체 검증, 네이버/토스/PASS/카카오톡 실제 인증 연동, Apple/Google 로그인 MVP 구현, passkey MVP 구현, 장기 KYC/AML

## 구조 분석

### 문서 역할

- `REQ.A.01`은 드롭 발견부터 구매 시도까지의 큰 사용자 여정을 다룬다.
- `REQ.A.05`는 그 여정에서 필요한 로그인, 회원가입, 세션, 사용자 계정 식별, 권한 확인을 독립 요구사항으로 분리한다.
- 공개 탐색 화면은 비회원 접근을 허용하고, 드롭 참여와 개인/결제 정보 화면은 인증 게이트를 통과해야 한다.
- 하나의 페이지 안에서도 공개 영역, 선택적 개인화 영역, 필수 인증 행동을 분리한다. 예를 들어 홈 화면은 공개 접근을 허용하되, 사용자 쿠폰/알림/최근 주문 같은 개인화 영역은 로그인 상태에서만 조회하거나 비회원 대체 상태로 표시한다.
- 휴대폰 번호 인증은 독립적인 사용자 계정 생성 수단이 아니라, 이메일 회원가입 과정에서 User Context가 발급한 `user_id`에 연결되는 인증 식별자다. 휴대폰 번호 로그인은 이미 연결된 휴대폰 인증 식별자를 검증한 뒤 해당 `user_id`로 로그인한다.
- 인증 서비스는 `user_id`와 인증/세션/권한 판단에 필요한 최소 정보만 다루고, 사용자 프로필과 표시 정보는 회원/사용자 서비스 책임으로 분리한다.

### 용어 구분

| 용어 | 의미 | 식별자/예시 | 원칙 |
| --- | --- | --- | --- |
| 사용자 계정 | DropMong 내부 사용자를 나타내는 주체이며 주문, 쿠폰, 알림, 권한, 감사 이력의 기준이 된다. | `user_id` | 사용자 계정끼리는 병합하지 않는다. |
| 인증 식별자 | 사용자가 로그인하거나 소유를 증명하는 구체적인 인증 식별자다. | 이메일 인증 식별자, 휴대폰 인증 식별자, Apple ProviderSubject, Google ProviderSubject | 하나의 인증 식별자는 하나의 `user_id`에만 연결된다. |
| 인증 수단 | 인증 식별자를 만들거나 검증하는 방식이다. | 이메일, 휴대폰 번호, Apple, Google, passkey | MVP와 후속 구현 범위를 분리한다. |
| 인증 수단 연동 | 현재 사용자 계정에 새 인증 식별자를 연결하는 작업이다. | `user_id` + 인증 식별자 | 이미 다른 사용자 계정에 연결된 인증 식별자는 연동할 수 없다. |

### MVP 범위 판단

- MVP에 포함: 이메일 회원가입, 이메일 회원가입 중 가상 SMS 기반 휴대폰 인증 식별자 연동, 이메일 로그인, 연동된 휴대폰 인증 식별자 기반 로그인, 검증된 휴대폰 번호 셀프 변경, 회원가입 후 자동 로그인, 기본 세션/refresh token 정책, 인증 게이트, 로그아웃, 감사 이벤트.
- 요구사항에만 반영: 네이버 인증, 토스 인증, PASS 인증, 카카오톡 인증, Apple 로그인, Google 로그인, 판매자/운영자 passkey, 사용자 선택 기반 인증 수단 연동.
- MVP에서 구현하지 않음: 네이버/토스/PASS/카카오톡 실제 인증 연동, Apple/Google 실제 연동, passkey 등록/인증, 복잡한 소셜 인증 수단 연동, 다국가 KYC.

### 근거 자료 구조

- `PAGE.A.300`은 로그인 메인, 이메일 회원가입, 이메일 로그인, 휴대폰 번호 로그인을 하나의 인증/회원 페이지 묶음으로 정의한다.
- `PAGE.A.310`은 비밀번호 찾기, 인증 방식 선택, 휴대폰 번호 인증, 이메일 인증, 새 비밀번호 설정으로 이어지는 비밀번호 재설정 과정 전체를 하나의 페이지 문서로 정의한다.
- `REQ.A.01`은 공개 탐색과 로그인 필요 행동의 경계를 이미 정의한다.
- Oliveyoung OAuth2 전환 사례는 인증 서버 장애, token refresh spike, feature flag, shadow mode, fallback의 필요성을 보여준다.
- Mercari Global Identity 사례는 사용자 계정 생성이 단일 폼이 아니라 identity, compatibility, signup orchestration 문제라는 점을 보여준다.
- Mercari passkey/OIDC 보안 사례는 고위험 사용자 계정에는 password/SMS보다 phishing-resistant 인증과 token trust model이 중요하다는 신호를 준다.
- 당근 고객센터 FAQ는 휴대폰 번호가 계정 접근 수단으로 쓰일 때 번호 변경과 기기 변경이 계정 복구 문의로 이어질 수 있다는 점을 보여준다.

## 문제 정의

- 사용자가 겪는 문제: 사용자는 드롭을 둘러보다가 알림 신청, 구매 시도, 주문 조회처럼 개인 정보가 필요한 순간에만 로그인하기를 원한다. 반대로 드롭 참여 시에는 신뢰 가능한 사용자 식별과 연락처 인증이 필요하다.
- 현재 방식의 한계: 로그인과 회원가입을 구매 흐름의 부속 입력값으로만 다루면 redirect 복귀, 세션 만료, 휴대폰 인증, 인증 수단 연동, 판매자/운영자 권한, 인증 장애 격리를 요구사항으로 설명하기 어렵다.
- 이 프로젝트가 해결해야 하는 것: DropMong은 비회원 탐색을 막지 않으면서도 드롭 참여, 쿠폰 발급, 주문/결제, 판매자 포털, 운영자 사이트에는 일관된 인증 게이트와 권한 검증을 적용해야 한다.
- 해결하지 않는 것: 모든 소셜 로그인과 passkey를 MVP에서 구현하지 않는다. 인증 수단 연동을 자동 추론으로 처리하지 않는다. 인증 서버 장애 자체를 없애지는 않지만 장애 영향을 격리하고 설명한다.

## 도출 근거

| 근거 | 확인한 신호 | DropMong 요구로 바꾼 내용 |
| --- | --- | --- |
| `REQ.A.01` 공개 탐색 요구 | 홈, 드롭 목록, 드롭 상세, 검색, 공지는 비회원 접근이 가능하고 구매 시도는 로그인으로 보낸다. | 인증 게이트는 화면/API 민감도별로 적용하고 로그인 후 원래 의도로 복귀해야 한다. |
| `PAGE.A.300` 로그인 메인 | 휴대폰, 이메일, Apple, Google 로그인 진입과 이메일 회원가입 링크를 가진다. | 지원 인증 수단은 설정으로 노출하고, 휴대폰 로그인은 연결된 휴대폰 인증 식별자가 있을 때만 허용한다. |
| `PAGE.A.301` 이메일 회원가입 | 이름, 이메일, 비밀번호, 휴대폰 번호, 추천인 코드, 약관 동의가 필요하다. | 회원가입 폼 검증, 필수 동의, 추천인 코드 검증, 가상 SMS 기반 휴대폰 인증 식별자 연동, 가입 후 자동 로그인을 요구사항으로 둔다. |
| `PAGE.A.302` 이메일 로그인 | 로그인 상태 유지, 비밀번호 재설정, 인증 식별자 잠금, 소셜 로그인 후보가 있다. | 세션 TTL과 refresh token 정책은 변경 가능한 설정으로 두고 기본값을 제공한다. |
| Oliveyoung OAuth2 전환 | 인증 서버 장애와 token refresh spike가 피크 트래픽에서 사용자 영향으로 이어질 수 있다. | 인증 서버 timeout, circuit breaker, token jitter, 단계적 전환, 관측 지표를 요구한다. |
| Mercari Global Identity | global account, legacy compatibility, signup orchestration이 사용자 계정 생성과 연결된다. | 사용자 식별자는 `user_id` 중심으로 관리하고 인증 수단 연동 상태 전이를 명확히 한다. |
| Mercari passkey/OIDC 보안 | passkey와 OIDC/OAuth threat modeling은 고가치 사용자 계정 보호에 중요하다. | 판매자와 운영자에는 passkey 요구사항을 두되 MVP 구현 범위에서는 제외한다. |
| [당근 고객센터 FAQ](https://cs.kr.karrotmarket.com/wv/faqs/32) | 휴대폰 번호가 계정 접근 수단이면 번호 변경과 기기 변경 시 기존 계정 접근 문제가 생길 수 있다. | 휴대폰 번호 변경은 안전한 로그인과 대체 인증을 통과한 경우 셀프 교체를 허용하고, 계정 접근이 불가능하면 CS/운영자 복구로 보낸다. |

## 페인포인트 개선 매핑

| 문제 ID | 페인포인트 | 개선 방향 | 연결 기능 요구사항 | 연결 비기능 요구사항 | 근거 |
| --- | --- | --- | --- | --- | --- |
| `REQ.A.05.PP-001` | 로그인 강제가 탐색 전환을 막는다. | 공개 탐색 화면은 비회원 접근을 허용하고, 개인/결제/드롭 참여 행동에서만 인증 게이트를 적용한다. | `REQ.A.05.FR-001`, `REQ.A.05.FR-002`, `REQ.A.05.FR-003` | `REQ.A.05.NFR-001` | [REQ.A.01](./REQ_A_01_limited_drop_commerce.md) |
| `REQ.A.05.PP-002` | 로그인 후 원래 행동으로 돌아가지 못하면 구매 전환이 끊긴다. | 로그인 진입 컨텍스트와 redirect target을 보존하고 성공 후 원래 화면/행동으로 복귀한다. | `REQ.A.05.FR-003`, `REQ.A.05.FR-009` | `REQ.A.05.NFR-002` | [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.PP-003` | 드롭 참여 시 사용자 계정만 있고 연락처 인증 정보가 없으면 공정성/CS 대응이 약해진다. | 이메일 회원가입 중 가상 SMS로 휴대폰 인증 식별자를 연결하고, 드롭 참여 전 사용자 인증 정보 보유 여부를 확인한다. | `REQ.A.05.FR-005`, `REQ.A.05.FR-010`, `REQ.A.05.FR-026` | `REQ.A.05.NFR-003`, `REQ.A.05.NFR-004` | [REQ.A.01](./REQ_A_01_limited_drop_commerce.md) |
| `REQ.A.05.PP-004` | 세션 만료 정책이 코드에 고정되면 피크/보안 정책 변경에 늦게 대응한다. | access token, refresh token, 로그인 상태 유지 TTL을 설정으로 변경 가능하게 하고 초기 기본값을 둔다. | `REQ.A.05.FR-011`, `REQ.A.05.FR-012` | `REQ.A.05.NFR-005`, `REQ.A.05.NFR-006` | [Oliveyoung OAuth2 전환](https://oliveyoung.tech/2025-10-28/oliveyoung-zero-downtime-oauth2-migration/) |
| `REQ.A.05.PP-005` | 인증 서버 장애가 드롭 구매 경로 전체로 번질 수 있다. | 인증 API timeout, circuit breaker, refresh token jitter, fallback/degraded mode 기준을 둔다. | `REQ.A.05.FR-011`, `REQ.A.05.FR-018` | `REQ.A.05.NFR-007`, `REQ.A.05.NFR-008` | [Oliveyoung OAuth2 전환](https://oliveyoung.tech/2025-10-28/oliveyoung-zero-downtime-oauth2-migration/) |
| `REQ.A.05.PP-006` | 이메일, 휴대폰, 소셜 인증 수단이 늘어나면 같은 사용자가 여러 인증 경로를 쓰고 싶어질 수 있다. | 사용자 계정을 병합하지 않고, 사용자가 선택한 기존 `user_id`에 새 인증 식별자를 연동한다. 이미 다른 `user_id`에 연결된 인증 식별자는 연동할 수 없으며, 고위험 인증 수단 변경은 셀프 변경 조건과 CS/운영자 수동 처리 조건을 나눈다. | `REQ.A.05.FR-016`, `REQ.A.05.FR-017`, `REQ.A.05.FR-018`, `REQ.A.05.FR-030` | `REQ.A.05.NFR-009`, `REQ.A.05.NFR-010`, `REQ.A.05.NFR-020` | [Mercari Global Identity](https://engineering.mercari.com/en/blog/entry/20251014-toward-a-global-identity-platform/) |
| `REQ.A.05.PP-007` | 판매자와 운영자 사용자 계정 탈취는 구매자 사용자 계정보다 영향 범위가 크다. | 판매자/운영자에는 passkey 요구사항을 두고, MVP 이후 강한 인증과 팀장급 승인 기반 복구 절차로 확장한다. | `REQ.A.05.FR-020`, `REQ.A.05.FR-021`, `REQ.A.05.FR-031` | `REQ.A.05.NFR-011`, `REQ.A.05.NFR-012`, `REQ.A.05.NFR-021` | [Mercari passkey](https://engineering.mercari.com/en/blog/entry/20251106-mercari-phishing-resistant-accounts-with-passkey/) |
| `REQ.A.05.PP-008` | OAuth/OIDC 토큰을 잘못 신뢰하면 인증과 권한 경계가 흐려진다. | ID token, access token, 내부 session token의 용도를 구분하고 token trust model을 문서화한다. | `REQ.A.05.FR-014`, `REQ.A.05.FR-015`, `REQ.A.05.FR-019` | `REQ.A.05.NFR-013`, `REQ.A.05.NFR-014` | [Mercari OIDC/OAuth Security](https://engineering.mercari.com/en/blog/entry/20251221-tales-of-oidc-oauth-security-what-it-takes-to-trust-a-token/) |
| `REQ.A.05.PP-009` | 홈 화면처럼 공개 페이지 안에도 사용자 관련 정보가 섞이면 전체 페이지를 로그인 전용으로 오해하기 쉽다. | 인증 정책을 페이지 단위뿐 아니라 영역, 행동, API 단위로 나누고 선택적 개인화 영역은 비회원 대체 상태를 제공한다. | `REQ.A.05.FR-001`, `REQ.A.05.FR-002`, `REQ.A.05.FR-023`, `REQ.A.05.FR-024` | `REQ.A.05.NFR-001`, `REQ.A.05.NFR-016`, `REQ.A.05.NFR-017` | [PAGE.A.01](../10-sitemap/PAGE_A_01_homepage.md) |
| `REQ.A.05.PP-010` | 사용자가 휴대폰 번호를 바꿨거나 기존 번호를 더 이상 쓸 수 없으면 휴대폰 로그인과 계정 복구가 CS 문의로 몰릴 수 있다. | 안전하게 로그인된 상태에서 이메일 재인증과 새 번호 SMS 인증을 통과하면 휴대폰 번호 셀프 변경을 허용하고, 대체 인증이 불가능하면 CS/운영자 복구로 보낸다. | `REQ.A.05.FR-030`, `REQ.A.05.FR-032`, `REQ.A.05.FR-033`, `REQ.A.05.FR-034` | `REQ.A.05.NFR-020`, `REQ.A.05.NFR-022` | [당근 고객센터 FAQ](https://cs.kr.karrotmarket.com/wv/faqs/32) |

## 목표

- 사용자 목표: 사용자는 불필요한 로그인 없이 드롭을 탐색하고, 필요한 순간에 빠르게 가입/로그인한 뒤 원래 행동으로 돌아간다.
- 비즈니스 목표: DropMong은 드롭 참여자와 주문 사용자를 신뢰 가능한 `user_id`로 식별해 공정성, CS 대응, 재참여 경험을 지킨다.
- 운영 목표: 운영자는 인증 실패, 인증 식별자 잠금, 세션 이상, 인증 수단 연동, 휴대폰 번호 변경, 권한 변경 이력을 감사 이벤트와 지표로 확인한다.
- 기술 목표: 인증/세션/권한 처리는 사용자 프로필과 분리하고, 피크 트래픽과 인증 서버 장애가 구매 경로 전체로 번지지 않게 한다.

## 사용자 유형

| Actor ID | 사용자 | 목표 | 주요 행동 |
| --- | --- | --- | --- |
| `ACTOR-GUEST` | 비회원 | 드롭을 둘러보고 필요한 순간 가입/로그인한다. | 홈/목록/상세 탐색, 로그인 수단 선택, 회원가입 |
| `ACTOR-BUYER` | 구매자 | 드롭 참여와 주문 확인에 필요한 인증 상태를 유지한다. | 로그인, 휴대폰 인증, 드롭 참여, 주문 조회, 로그아웃 |
| `ACTOR-SELLER` | 판매자 | 판매자 포털에 안전하게 접근한다. | 판매자 로그인, 권한 확인, passkey 등록 예정 |
| `ACTOR-PLATFORM-OPERATOR` | 플랫폼 운영자 | 운영자 사이트에 강한 인증으로 접근한다. | 운영자 로그인, RBAC 확인, passkey 등록 예정 |
| `ACTOR-CS` | CS 담당자 | 사용자 계정과 인증 식별자 상태를 필요한 범위에서 확인한다. | 사용자 계정 상태 조회, 인증 식별자 잠금/실패 이력 확인, 인증 수단 연동 문의 확인, 번호 변경 복구 문의 처리 |

## 기능 요구사항

| Req ID | 요구사항 | 사용자 | 우선순위 | 연결 Page/UC |
| --- | --- | --- | --- | --- |
| `REQ.A.05.FR-001` | 사용자는 로그인하지 않아도 홈, 드롭 목록, 드롭 상세, 검색, 공지처럼 개인 정보나 결제 정보가 필요 없는 화면을 탐색할 수 있다. | 비회원, 구매자 | Must | [PAGE.A.01](../10-sitemap/PAGE_A_01_homepage.md), [PAGE.A.02](../10-sitemap/PAGE_A_02_product_detail.md) |
| `REQ.A.05.FR-002` | 시스템은 비회원이 알림 신청, 관심, 장바구니, 바로 구매, 마이, 주문 내역, 결제수단처럼 개인 정보나 드롭 참여가 필요한 행동을 시도하면 로그인 페이지로 이동시킨다. | 비회원 | Must | [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-003` | 시스템은 로그인 진입 시 원래 가려던 화면과 의도한 행동을 보존하고, 로그인 성공 후 해당 화면 또는 행동으로 복귀시킨다. | 비회원, 구매자 | Must | [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-004` | 사용자는 이메일 주소, 비밀번호, 이름, 휴대폰 번호, 추천인 코드, 약관 동의를 입력해 이메일 회원가입을 요청한다. | 비회원 | Must | [PAGE.A.301](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-005` | 사용자는 이메일 회원가입 중 가상 SMS 휴대폰 인증을 완료해 User Context가 발급하는 `user_id`에 휴대폰 인증 식별자를 연결한다. | 비회원 | Must | [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-006` | 시스템은 이메일 형식, 이메일 중복, 비밀번호 규칙, 비밀번호 일치, 필수 약관 동의, 휴대폰 번호 형식, 가상 SMS 인증번호를 검증한 뒤 회원가입을 처리한다. | 비회원 | Must | [PAGE.A.301](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-007` | 시스템은 이메일 인증 메일 검증과 가상 SMS 휴대폰 인증이 완료되어 회원가입에 성공하면 User Context가 발급한 새 `user_id`에 인증 식별자를 연결하고 사용자를 자동 로그인 상태로 전환한다. | 비회원 | Must | [PAGE.A.301](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-008` | 사용자는 이메일과 비밀번호로 로그인하고, 필요하면 로그인 상태 유지를 선택한다. | 비회원, 구매자 | Must | [PAGE.A.302](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-009` | 시스템은 로그인 성공 후 redirect target이 있으면 해당 위치로 이동하고, 없으면 홈 화면으로 이동한다. | 구매자 | Must | [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md), [PAGE.A.302](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-010` | 시스템은 드롭 참여 전 로그인 상태, 사용자 인증 정보 보유 여부, 구매 제한 조건, 차단된 사용자 계정 여부를 확인하고 진행 가능 여부를 결정한다. | 구매자 | Must | [REQ.A.01](./REQ_A_01_limited_drop_commerce.md) |
| `REQ.A.05.FR-011` | 시스템은 access token, refresh token, 로그인 상태 유지 세션을 발급하고 만료 시각을 함께 관리한다. | 구매자 | Must | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |
| `REQ.A.05.FR-012` | 운영자는 access token TTL, refresh token TTL, 로그인 상태 유지 TTL, refresh rotation 여부를 배포 없이 설정으로 변경할 수 있다. | 플랫폼 운영자 | Must | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |
| `REQ.A.05.FR-013` | 시스템은 초기 기본값으로 access token 15분, refresh token 14일, 로그인 상태 유지 refresh token 30일, refresh rotation 사용을 제공한다. | 시스템 | Must | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |
| `REQ.A.05.FR-014` | 사용자는 로그아웃할 수 있고, 시스템은 해당 세션 또는 refresh token을 더 이상 사용할 수 없게 만든다. | 구매자 | Must | PAGE.A.10 예정 |
| `REQ.A.05.FR-015` | 시스템은 로그인 실패, 인증 식별자 잠금, 비밀번호 재설정 필요, 인증 수단 비활성 같은 실패 사유를 사용자와 CS가 이해할 수 있는 코드로 남긴다. | 구매자, CS | Must | [PAGE.A.302](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-016` | 사용자는 현재 로그인한 사용자 계정의 `user_id`에 이메일, 휴대폰 번호, Apple, Google 같은 추가 인증 식별자 연동을 요청할 수 있다. | 구매자 | Should | 사용자 계정 관리 페이지 예정 |
| `REQ.A.05.FR-017` | 시스템은 인증 수단 연동 시 해당 인증 식별자가 이미 다른 `user_id`에 등록되어 있으면 연동을 거부하고 사유 코드를 반환한다. | 구매자, CS | Should | 사용자 계정 관리 페이지 예정 |
| `REQ.A.05.FR-018` | 시스템은 인증 수단 연동 요청 시 현재 사용자 계정의 `user_id`, 연동할 인증 식별자, 소유 증명 방식, 연동 성공/실패 결과를 기록한다. | 구매자, CS | Should | 사용자 계정 관리 페이지 예정 |
| `REQ.A.05.FR-019` | 시스템은 네이버 인증, 토스 인증, PASS 인증, 카카오톡 인증, Apple 로그인, Google 로그인을 인증 수단 후보로 요구사항에 반영하되 MVP 구현 범위에서는 제외한다. | 비회원, 구매자 | Could | [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-020` | 판매자 사용자 계정은 판매자 포털 접근 시 일반 구매자 권한과 분리된 역할 확인을 통과해야 한다. | 판매자 | Must | [REQ.A.03](./REQ_A_03_seller.md) |
| `REQ.A.05.FR-021` | 운영자 사용자 계정은 운영자 사이트 접근 시 RBAC를 통과해야 하고, 판매자/운영자 passkey는 MVP 이후 강한 인증 요구사항으로 둔다. | 플랫폼 운영자 | Must | [REQ.A.04](./REQ_A_04_platform_operator_admin.md) |
| `REQ.A.05.FR-022` | 시스템은 로그인 성공/실패, 회원가입, 휴대폰 인증, token 발급/갱신, 로그아웃, 인증 식별자 잠금, 권한 변경, 인증 수단 연동/변경을 의미별 감사 이벤트로 구분해 Audit Context로 전송한다. | 시스템, CS, 운영자 | Must | 운영자 사이트 예정 |
| `REQ.A.05.FR-023` | 시스템은 화면 영역과 사용자 행동을 공개, 선택적 인증, 필수 인증, 권한 필요로 분류하고 각 분류별 처리 방식을 정의한다. | 시스템 | Must | PAGE.A.01, PAGE.A.300 예정 |
| `REQ.A.05.FR-024` | 공개 페이지 안의 사용자 개인화 컴포넌트는 Auth Boundary 또는 Permission Boundary 안에서 렌더링하며, 인증/권한 오류가 발생해도 페이지 전체 실패로 전파하지 않고 해당 컴포넌트의 대체 상태로 제한한다. | 비회원, 구매자 | Must | [PAGE.A.01](../10-sitemap/PAGE_A_01_homepage.md) |
| `REQ.A.05.FR-025` | API는 공개 데이터, 선택적 개인화 데이터, 필수 인증 데이터를 구분해 제공하고, 선택적 개인화 데이터 실패가 공개 페이지 전체 실패로 번지지 않게 한다. | 시스템 | Must | [SD.A.30040](../50-service-design/A_300_auth/A_300_40-api/README.md) |
| `REQ.A.05.FR-026` | 시스템은 휴대폰 번호 로그인 시 가상 SMS 인증을 통과한 휴대폰 인증 식별자가 기존 `user_id`에 연결되어 있을 때만 해당 사용자 계정으로 로그인시킨다. 연결된 `user_id`가 없으면 로그인 실패와 이메일 회원가입/연동 안내를 반환한다. | 비회원, 구매자 | Must | [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-027` | 시스템은 이메일 인증 메일 검증을 이메일 회원가입 완료의 필수 조건으로 둔다. 휴대폰 인증 식별자가 연결되어 있어도 이메일 인증을 생략하지 않는다. | 비회원 | Must | [PAGE.A.301](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-028` | 사용자는 비밀번호 재설정을 이메일 인증과 휴대폰 번호 인증 중 하나로 요청할 수 있다. | 비회원, 구매자 | Must | [PAGE.A.310](../10-sitemap/PAGE_A_310_password_find/PAGE_A_310_password_find.md) |
| `REQ.A.05.FR-029` | 시스템은 동일 인증 식별자의 로그인 실패가 5회에 도달하면 전역 잠금 정책을 적용하고, 잠금 상태와 해제 가능 시점을 사용자와 CS가 이해할 수 있는 코드로 남긴다. | 비회원, 구매자, CS | Must | [PAGE.A.302](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.FR-030` | 시스템은 인증 수단 해제와 재연동을 고위험 인증 수단 변경으로 다룬다. 안전하게 로그인되어 있고 대체 인증 수단으로 재인증한 사용자는 허용된 범위에서 셀프 변경할 수 있으며, 계정 접근이 불가능하거나 대체 인증이 불가능하면 CS/플랫폼 운영자 수동 처리로 보낸다. | 구매자, CS, 플랫폼 운영자 | Must | 사용자 계정 관리 페이지 예정, 운영자 사이트 예정 |
| `REQ.A.05.FR-031` | 판매자/운영자 passkey 도입 후 복구용 인증 식별자 변경과 break-glass 권한 사용은 팀장급 승인자를 거쳐야 한다. | 판매자, 플랫폼 운영자 | Must | 운영자 사이트 예정 |
| `REQ.A.05.FR-032` | 사용자는 이메일 로그인 또는 이메일 재인증을 통과한 로그인 상태에서 새 휴대폰 번호의 가상 SMS 인증을 완료해 기존 `user_id`에 연결된 휴대폰 인증 식별자를 새 번호로 교체할 수 있다. | 구매자 | Must | 사용자 계정 관리 페이지 예정 |
| `REQ.A.05.FR-033` | 시스템은 휴대폰 번호 변경 시 기존 휴대폰 인증 식별자를 즉시 삭제하지 않고 `replaced`, `revoked`, `superseded` 같은 닫힘 상태로 전환한 뒤 새 휴대폰 인증 식별자를 같은 `user_id`에 연결한다. | 시스템, CS | Must | [SD.A.30020](../50-service-design/A_300_auth/A_300_20-persistence/README.md) |
| `REQ.A.05.FR-034` | 시스템은 휴대폰 번호 변경을 사용자 계정 병합으로 처리하지 않고, `identity_id`와 `user_id`의 연결 교체로 처리한다. | 시스템, CS | Must | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |

## 비기능 요구사항

| Req ID | 요구사항 | 기준 | 연결 문서 |
| --- | --- | --- | --- |
| `REQ.A.05.NFR-001` | 인증 게이트는 화면과 API의 정보 민감도에 맞춰 일관되게 적용해야 한다. | 공개 탐색 화면은 인증 없이 응답하고, 개인/결제/드롭 참여 API는 인증 실패 시 로그인 이동 또는 `401`을 반환한다. | [REQ.A.01](./REQ_A_01_limited_drop_commerce.md) |
| `REQ.A.05.NFR-002` | redirect target은 변조되거나 외부 악성 URL로 사용되지 않아야 한다. | 로그인 후 복귀 위치는 허용된 내부 경로와 검증된 intent id만 사용한다. | [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.NFR-003` | 드롭 참여 가능 여부는 로그인 상태만으로 판단하지 않아야 한다. | 로그인 세션, 사용자 인증 정보, 차단 상태, 구매 제한 조건을 함께 확인한다. | [REQ.A.01](./REQ_A_01_limited_drop_commerce.md) |
| `REQ.A.05.NFR-004` | 휴대폰 번호 인증 정보는 최소 보관과 암호화를 기본값으로 한다. | 원문 노출을 제한하고, 조회/변경/검증 이력은 의미별 감사 이벤트로 전송한다. | [SD.A.30020](../50-service-design/A_300_auth/A_300_20-persistence/README.md) |
| `REQ.A.05.NFR-005` | token과 세션 만료 정책은 설정으로 변경 가능해야 한다. | TTL, rotation, remember-me 정책 변경에 코드 배포가 필요하지 않다. | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |
| `REQ.A.05.NFR-006` | refresh token은 재사용과 탈취를 탐지할 수 있어야 한다. | refresh rotation 사용 시 이전 token 재사용을 이상 징후로 기록하고 세션을 폐기할 수 있다. | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |
| `REQ.A.05.NFR-007` | 인증 서버 장애는 드롭 구매 경로 전체로 전파되지 않아야 한다. | 인증 API timeout, circuit breaker, 제한된 degraded mode, 사용자 안내 기준을 둔다. | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |
| `REQ.A.05.NFR-008` | 로그인과 token 갱신은 피크 트래픽에서 관측 가능해야 한다. | signin, signup, token refresh, phone verification의 p50/p95/p99 latency, error rate, lock rate, refresh spike를 확인한다. | 운영 대시보드 예정 |
| `REQ.A.05.NFR-009` | 사용자 계정은 자동 또는 수동으로 병합되지 않아야 한다. | 이메일, 휴대폰, ProviderSubject가 같거나 달라도 기존 `user_id`끼리 합치지 않고, 인증 수단 연동만 별도 상태 전이로 처리한다. | 사용자 계정 관리 페이지 예정 |
| `REQ.A.05.NFR-010` | 인증 식별자는 하나의 `user_id`에만 연결될 수 있어야 한다. | 이미 다른 `user_id`에 연결된 이메일 인증 식별자, 휴대폰 인증 식별자, ProviderSubject는 새 `user_id`에 연동할 수 없고 실패 사유와 감사 이벤트를 남긴다. | [SD.A.30020](../50-service-design/A_300_auth/A_300_20-persistence/README.md) |
| `REQ.A.05.NFR-011` | 판매자와 운영자 사용자 계정은 일반 구매자 사용자 계정보다 강한 인증을 요구할 수 있어야 한다. | passkey, MFA, 재인증 정책을 역할별로 설정할 수 있다. | [REQ.A.03](./REQ_A_03_seller.md), [REQ.A.04](./REQ_A_04_platform_operator_admin.md) |
| `REQ.A.05.NFR-012` | passkey는 MVP 구현 범위에서 제외하되 요구사항과 확장 경로는 남긴다. | 판매자/운영자 passkey 등록, 복구, 해제, 분실 대응 정책을 후속 문서에서 다룰 수 있다. | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |
| `REQ.A.05.NFR-013` | OAuth/OIDC token은 용도별로 구분해야 한다. | ID token은 identity proof, access token은 권한 위임, 내부 session token은 DropMong API 접근에 사용한다. | [SD.A.30040](../50-service-design/A_300_auth/A_300_40-api/README.md) |
| `REQ.A.05.NFR-014` | 인증 서비스는 사용자 프로필 의미를 알지 않아야 한다. | auth-service는 `user_id`, credential, session, role/permission claim만 다루고 표시명, 주소, 마케팅 속성은 회원/사용자 서비스가 다룬다. | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |
| `REQ.A.05.NFR-015` | 인증 관련 감사 이벤트는 민감 정보를 포함하지 않아야 한다. | 비밀번호, 인증번호, token 원문, provider secret은 로그/trace/event에 남기지 않는다. | 운영 대시보드 예정 |
| `REQ.A.05.NFR-016` | 인증 정책은 페이지 단위에만 고정되지 않아야 한다. | page, section, action, API endpoint 단위로 공개, 선택적 인증, 필수 인증, 권한 필요 정책을 선언할 수 있다. | PAGE.A.01, [SD.A.30040](../50-service-design/A_300_auth/A_300_40-api/README.md) |
| `REQ.A.05.NFR-017` | 선택적 개인화 API의 인증 실패는 공개 페이지 전체를 실패시키지 않아야 한다. | 홈 화면 공개 데이터는 유지하고, Auth Boundary 또는 Permission Boundary 안의 개인화 컴포넌트만 비회원 상태, 로그인 유도, 권한 없음, 또는 일시 실패 상태로 처리한다. | [PAGE.A.01](../10-sitemap/PAGE_A_01_homepage.md) |
| `REQ.A.05.NFR-018` | 이메일 인증은 휴대폰 인증보다 낮은 우선순위의 선택 절차가 아니어야 한다. | 이메일 회원가입은 이메일 인증 메일 검증과 가상 SMS 휴대폰 인증 식별자 연동을 모두 만족해야 완료된다. | [PAGE.A.301](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md) |
| `REQ.A.05.NFR-019` | 로그인 실패 횟수와 인증 식별자 잠금 시간은 전역 정책으로 관리해야 한다. | 기본 실패 허용 횟수는 5회로 두고, 실패 횟수 기준과 잠금 시간은 배포 없이 설정으로 조정할 수 있다. | [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md) |
| `REQ.A.05.NFR-020` | 인증 수단 해제와 재연동은 기본 셀프서비스로 열지 않아야 한다. | 안전한 로그인, 대체 인증, 새 번호 SMS 인증을 만족한 휴대폰 번호 교체만 셀프 변경으로 허용하고, 그 외 고위험 변경은 CS/운영자 수동 처리로 제한한다. | 운영자 사이트 예정 |
| `REQ.A.05.NFR-021` | passkey 복구와 break-glass 권한은 고위험 운영 절차로 취급해야 한다. | 팀장급 승인자 기록, 사유, 만료 시각, 사용 결과, 사후 검토 상태를 감사 이벤트로 남긴다. | 운영자 사이트 예정 |
| `REQ.A.05.NFR-022` | 휴대폰 번호 변경은 감사와 복구가 가능해야 한다. | 기존 휴대폰 인증 식별자의 닫힘 상태, 새 휴대폰 인증 식별자, 대상 `user_id`, 변경 경로, 재인증 결과, 감사 이벤트 전송 결과를 추적할 수 있어야 한다. | [SD.A.30020](../50-service-design/A_300_auth/A_300_20-persistence/README.md), Audit Context 예정 |

## 제약 조건

- 정책/법적 제약: 개인정보 처리, 만 14세 이상 동의, 약관 동의, 마케팅 수신 동의, 휴대폰 번호 보관/마스킹, 감사 이벤트 보존 기준을 준수해야 한다.
- 기술 제약: 드롭 오픈 순간에는 로그인, token refresh, 휴대폰 인증 요청이 동시에 증가할 수 있으므로 구매/쿠폰 핵심 쓰기 경로와 인증 경로의 장애 영향을 분리해야 한다.
- 일정 제약: MVP는 이메일 회원가입과 가상 SMS 휴대폰 인증 식별자 연동을 우선 구현하고, 네이버/토스/PASS/카카오톡 인증, Apple/Google 로그인, 판매자/운영자 passkey는 요구사항에만 반영한다.
- 데이터 제약: `user_id`는 사용자 계정의 내부 식별자이며 회원, 주문, 쿠폰, 판매자, 운영자 감사 이력을 연결한다. 이메일 인증 식별자, 휴대폰 인증 식별자, 소셜 ProviderSubject는 서로 달라도 되지만, 각 인증 식별자는 한 `user_id`에만 연결된다.
- 운영 제약: 인증 수단 해제/재연동, 인증 식별자 잠금 해제, 권한 변경, passkey 해제 같은 작업은 수동 DB 수정이 아니라 CS/플랫폼 운영자의 운영 기능과 감사 이벤트를 통해 수행해야 한다. 다만 휴대폰 번호 교체는 안전한 로그인과 대체 인증, 새 번호 SMS 인증을 통과한 경우에만 사용자 셀프 변경을 허용한다.

## 수용 기준

- 비회원은 홈, 드롭 목록, 드롭 상세, 검색, 공지 화면을 볼 수 있다.
- 홈 화면의 공개 영역은 비회원에게 표시되고, 사용자 쿠폰/알림/최근 주문 같은 개인화 영역은 로그인 상태에서만 실제 사용자 데이터를 표시한다.
- 선택적 개인화 컴포넌트에서 인증 정보가 없거나 만료되어도 홈 화면 전체는 실패하지 않고 Auth Boundary 또는 Permission Boundary 안에서 비회원 대체 상태를 표시한다.
- 비회원이 알림 신청, 바로 구매, 장바구니, 마이, 주문 내역, 결제수단에 접근하면 로그인 페이지로 이동한다.
- 로그인 성공 후 redirect target이 있으면 원래 화면 또는 행동으로 돌아간다.
- 이메일 회원가입은 필수 입력과 필수 약관 동의를 통과해야 완료된다.
- 이메일 인증 메일 검증은 이메일 회원가입 필수 조건이다.
- 이메일 회원가입 중 가상 SMS 휴대폰 인증을 완료하면 휴대폰 인증 식별자가 User Context가 발급한 새 `user_id`에 연결된다.
- 휴대폰 번호 로그인은 이미 `user_id`에 연결된 휴대폰 인증 식별자가 있을 때만 성공한다.
- 비밀번호 재설정은 이메일 인증과 휴대폰 번호 인증을 모두 지원한다.
- 로그인 실패 5회가 발생하면 전역 인증 식별자 잠금 정책이 적용된다.
- 회원가입 성공 후 사용자는 자동 로그인 상태가 된다.
- 이메일 로그인은 로그인 상태 유지 옵션을 제공한다.
- access token, refresh token, 로그인 상태 유지 TTL은 설정으로 변경 가능하고 초기 기본값을 가진다.
- 드롭 참여 전에는 로그인 상태와 사용자 인증 정보 보유 여부를 확인한다.
- 네이버/토스/PASS/카카오톡 인증과 Apple/Google 로그인 버튼 또는 요구사항은 남기되 MVP 구현 대상에서 제외된다.
- 사용자 계정은 병합되지 않는다.
- 사용자는 현재 사용자 계정의 `user_id`에 추가 인증 식별자를 연동할 수 있다.
- 이미 다른 `user_id`에 등록된 이메일 인증 식별자, 휴대폰 인증 식별자, 소셜 ProviderSubject는 현재 사용자 계정에 연동할 수 없다.
- 인증 수단 해제와 재연동은 고위험 변경으로 분류된다.
- 안전하게 로그인된 사용자는 이메일 재인증과 새 휴대폰 번호 SMS 인증을 통과한 경우 휴대폰 번호를 셀프 변경할 수 있다.
- 휴대폰 번호 변경은 기존 `user_id`에 연결된 휴대폰 인증 식별자 교체로 처리하고 사용자 계정 병합으로 처리하지 않는다.
- 기존 휴대폰 인증 식별자는 닫힘 상태로 남고 새 휴대폰 인증 식별자가 같은 `user_id`에 연결된다.
- 계정 접근이 불가능하거나 대체 인증을 통과할 수 없으면 CS 담당자와 플랫폼 운영자의 운영 기능으로 수동 처리한다.
- 판매자와 운영자 사용자 계정에는 passkey 요구사항을 남기되 MVP 구현 대상에서 제외된다.
- 판매자/운영자 passkey 복구와 break-glass 권한 사용은 팀장급 승인자를 거쳐야 한다.
- 인증 서비스는 사용자 프로필/표시 정보를 소유하지 않고 `user_id`와 인증/세션/권한 정보만 다룬다.
- 로그인 성공/실패, 회원가입, token 갱신, 로그아웃, 인증 식별자 잠금, 인증 수단 연동/변경은 의미별 감사 이벤트로 Audit Context에 전송된다.

## 연관 태그

- 플로우 참조: FLOW.A.300, FLOW.A.301, FLOW.A.302, FLOW.A.303, FLOW.A.310
- 페이지 참조: [PAGE.A.300](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md), [PAGE.A.310](../10-sitemap/PAGE_A_310_password_find/PAGE_A_310_password_find.md)
- UI 참조: [UI.A.300](../20-ui/UI_A_300_auth_member/UI_A_300_auth_member.md), [UI.A.310](../20-ui/UI_A_310_password_find/UI_A_310_password_find.md)
- UC 참조: [UC.A.300](../30-uc/UC_A_300_auth_member.md)
- 도메인 참조: [SD.A.30010](../50-service-design/A_300_auth/A_300_10-domain-model/SD_A_30010_auth_domain_model.md)
- 영속성 참조: [SD.A.30020](../50-service-design/A_300_auth/A_300_20-persistence/README.md)
- 서비스 참조: [SD.A.30030](../50-service-design/A_300_auth/A_300_30-service/README.md)
- API 참조: [SD.A.30040](../50-service-design/A_300_auth/A_300_40-api/README.md)
- 시나리오 참조: SCN.A.300 예정, SCN.A.310 예정

## 열린 질문

- 휴대폰 번호 셀프 변경이 실패했거나 대체 인증을 통과할 수 없는 사용자를 CS/운영자 복구로 넘기는 세부 기준을 정해야 한다.

## 확인 필요

- 초기 세션 기본값: access token 15분, refresh token 14일, 로그인 상태 유지 30일이 DropMong MVP 운영 정책에 맞는지 확인한다.
- 가상 SMS 인증의 발송 제한, 인증번호 TTL, 재시도 제한, 테스트 번호 정책을 정한다.
- 약관/개인정보처리방침/마케팅 수신 동의의 실제 문서 URL과 버전 관리 방식을 정한다.
- 인증 수단 연동의 소유 증명 방식과 CS/플랫폼 운영자 수동 처리 화면의 승인 필드를 정한다.
- 휴대폰 번호 셀프 변경에서 허용할 대체 인증 수단, 재인증 TTL, 실패 횟수, 이전 번호 닫힘 상태 값을 정한다.
- Apple/Google 로그인의 후속 구현 시 ProviderSubject 저장 방식과 IdentityLink 정책을 정한다.
- 판매자/운영자 passkey의 후속 구현 시 등록, 분실, 복구, 강제 해제, 팀장급 승인 기록, 감사 이벤트 기준을 정한다.

## 새로 드러난 가장자리

- 이 자료가 답한 질문: DropMong 인증은 구매 플로우의 보조 입력이 아니라, 비회원 탐색과 드롭 참여 가능 여부를 나누는 독립 요구사항이다.
- 아직 남은 질문: 휴대폰 번호 셀프 변경 실패 또는 대체 인증 불가 상태를 CS/운영자 복구로 넘기는 기준이 남아 있다.
- 검증되지 않은 전제: MVP 사용자는 이메일 사용자 계정과 연결된 가상 SMS 휴대폰 인증 식별자로 드롭 참여에 필요한 인증 수준을 충족한다고 가정했다.
- 약한 연결: 인증 수단 연동은 credential 저장소, 소유 증명, 감사 이벤트, CS 조회 화면과 맞물리므로 후속 persistence/service 문서가 필요하다.
- 다음에 확인할 것: 가상 SMS 인증번호 정책, 세션 기본값, 인증 수단 연동 상태 전이, 휴대폰 번호 셀프 변경 실패 처리, CS/플랫폼 운영자 수동 처리 화면, 판매자/운영자 passkey 도입 시점

## 레퍼런스

- DropMong: [REQ.A.01 한정 상품 드롭 커머스](./REQ_A_01_limited_drop_commerce.md)
- DropMong: [PAGE.A.300 인증 및 회원 페이지](../10-sitemap/PAGE_A_300_auth_member/PAGE_A_300_auth_member.md)
- DropMong: [PAGE.A.310 비밀번호 재설정 페이지](../10-sitemap/PAGE_A_310_password_find/PAGE_A_310_password_find.md)
- Oliveyoung: [올리브영 무중단 OAuth2 마이그레이션](https://oliveyoung.tech/2025-10-28/oliveyoung-zero-downtime-oauth2-migration/)
- Mercari: [Toward a Global Identity Platform](https://engineering.mercari.com/en/blog/entry/20251014-toward-a-global-identity-platform/)
- Mercari: [Mercari's Phishing-Resistant Accounts with Passkey](https://engineering.mercari.com/en/blog/entry/20251106-mercari-phishing-resistant-accounts-with-passkey/)
- Mercari: [Tales of OIDC & OAuth Security](https://engineering.mercari.com/en/blog/entry/20251221-tales-of-oidc-oauth-security-what-it-takes-to-trust-a-token/)
- 당근: [고객센터 FAQ](https://cs.kr.karrotmarket.com/wv/faqs/32)
