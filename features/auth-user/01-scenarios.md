---
title: 인증과 사용자 시나리오
status: draft
created: 2026-07-03
updated: 2026-07-06
owner: TBD
scope: auth-user
---

# 인증과 사용자 시나리오

## 1. 목적

인증 계정과 사용자 계정을 분리한 상태에서 사용자가 여러 인증 수단으로 같은 사용자 계정에 접근할 수 있는 사용자 시나리오를 정의한다.

이 문서는 구현 세부사항보다 사용자 관점의 요구사항과 흐름을 먼저 확정한다.

## 검토 질문

1. 이메일 사용자 가입 시 사용자는 어떤 정보를 입력해야 하나요? ([FR-013](#fr-013-이메일-가입-최소-입력-정보))
2. 이메일/비밀번호 가입에서 이메일 소유 검증은 MVP에 포함하나요? ([FR-014](#fr-014-이메일-소유-검증-목킹))
3. OAuth/OIDC 최초 로그인 시 동일 이메일 계정이 있으면 사용자에게 어떤 선택지를 보여주나요? ([FR-001](#fr-001-oauth-동일-이메일-계정-처리))
4. OAuth/OIDC 제공자가 이메일을 제공하지 않거나 `email_verified`를 제공하지 않으면 가입을 허용하나요?
5. 비밀번호 찾기와 재설정은 MVP에서 제외해도 되나요? ([FR-004](#fr-004-비밀번호-찾기-제외))
6. 로그아웃은 현재 세션만 종료해도 되나요? ([FR-005](#fr-005-로그아웃-범위))
7. 비로그인 사용자가 접근 가능한 기능은 각 피처가 정하도록 둘까요? ([FR-006](#fr-006-비로그인-공개-기능-범위))
8. 사용자 탈퇴 또는 계정 비활성화는 MVP에 포함하나요? ([FR-010](#fr-010-사용자-탈퇴와-계정-비활성화-제외))
9. 인증 수단 연결 해제는 MVP에 포함하나요? ([FR-011](#fr-011-인증-수단-연결-해제-제외))
10. 약관, 개인정보, 마케팅, 알림 동의는 가입 시점에 받을까요, 구매 또는 알림 설정 시점에 받을까요? ([FR-012](#fr-012-가입-시-동의-정보-제외))

## 2. 사용자 흐름

### 2.1 전제

- 인증은 사용자가 누구인지 증명하는 과정이다.
- 사용자 계정은 DropMong 서비스 안에서 구매, 주문, 알림 상태를 가진다.
- 인증 수단마다 auth-service의 `AuthAccount`가 하나씩 존재한다.
- 하나의 사용자는 이메일 로그인, OAuth 간편 로그인 등 여러 `AuthAccount`와 `user_id`로 연결될 수 있다.
- 외부 인증 제공자는 신원 확인 수단으로만 사용하고, 서비스 내부 사용자 식별자는 DropMong이 소유한다.
- 인증 결과로부터 role 기반 인가에 사용할 RBAC 정보를 만든다.
- 인가는 인증 결과, role, 리소스 소유권, auth-service의 인가 차단 정책을 함께 사용해 판단한다.
- 사용자 상태는 user-service만 관리한다. 인가 차단은 사용자 상태를 직접 읽지 않고 auth-service의 role/ACL 정책으로 처리한다.
- RBAC만으로 표현하기 어려운 예외 권한은 ACL override로 보완할 수 있어야 한다.
- 웹과 모바일은 같은 access JWT와 opaque refresh token 구조를 사용한다.
- Istio IngressGateway 이후 내부 서비스가 받는 사용자 식별 헤더는 `X-User-*`로 통일한다.
- 서비스 내부 인가 판단이 필요한 API는 `X-User-*` 헤더를 애플리케이션 레벨의 Principal 또는 AuthContext로 변환한 뒤 수행한다.
- 실사용자에게 게스트 로그인을 제공하지 않는다.
- 로그인하지 않은 사용자는 익명 권한으로 접근 가능한 공개 기능만 사용할 수 있다.
- 개발 빌드에서는 인증 서비스가 테스트용 토큰 기반 계정 발급 API를 제공할 수 있다.

### 2.2 인증 수단

- [x] 이메일/비밀번호 로그인: 로컬 개발과 기본 로그인 수단으로 사용한다.
- [x] OAuth 간편 로그인: Google, Kakao, Apple, Microsoft 같은 외부 인증 제공자를 사용한다.
- [x] OIDC 로그인: OAuth 기반 로그인에서 표준 신원 claim을 검증한다.
- [x] 테스트 계정 토큰 발급: 내부 개발과 QA에서만 사용하는 인증 수단이다.
- [ ] 익명 접근 권한: 로그인하지 않은 사용자가 공개 기능을 사용할 수 있는 인가 정책이다.
- [ ] Passkey/WebAuthn: 비밀번호 없이 기기 생체인증, PIN, 보안키를 사용한다.
- [ ] Magic Link: 이메일 로그인 링크로 인증한다.
- [ ] OTP: SMS, 이메일, TOTP 앱 기반 일회용 코드를 사용한다.
- [ ] MFA/2FA: 비밀번호, OTP, Passkey 등을 조합해 추가 인증을 요구한다.
- [ ] SAML SSO: 기업 또는 운영자 계정용 SSO 연동에 사용한다.
- [ ] Device Login: TV, 콘솔, 키오스크처럼 입력이 불편한 기기에서 코드 기반으로 인증한다.

### 2.3 액터

| 액터 | 설명 |
| --- | --- |
| 고객 | 상품을 탐색하고 구매하는 사용자 |
| 판매자 | 상품과 드롭을 준비하고 판매 상태를 확인하는 사용자 |
| 운영자 | 서비스 운영, 장애 대응, 계정/권한 관리를 수행하는 사용자 |

### 2.4 유스케이스

```mermaid
flowchart LR
    subgraph CustomerArea["고객 유스케이스"]
        direction TB
        Customer["고객"]
        AnonymousAccess["비로그인 공개 기능 접근"]
        EmailSignup["이메일 기반 사용자 가입"]
        EmailLogin["이메일 기반 로그인"]
        OAuthLogin["OAuth/OIDC 간편 로그인"]
        LinkAuth["새 인증 수단 연결"]
        ViewMe["내 정보 확인"]
        Logout["로그아웃"]

        Customer --> AnonymousAccess
        Customer --> EmailSignup
        Customer --> EmailLogin
        Customer --> OAuthLogin
        Customer --> LinkAuth
        Customer --> ViewMe
        Customer --> Logout
    end

    subgraph SellerArea["판매자 유스케이스"]
        direction TB
        Seller["판매자"]
        SellerLogin["판매자 로그인"]
        SellerMe["판매자 내 정보 확인"]
        SellerLogout["판매자 로그아웃"]

        Seller --> SellerLogin
        Seller --> SellerMe
        Seller --> SellerLogout
    end

    subgraph OperatorArea["운영자 유스케이스"]
        direction TB
        Operator["운영자"]
        OperatorLogin["운영자 로그인"]
        IssueTestToken["테스트 계정 토큰 발급"]
        ManageRoles["사용자/판매자 권한 관리"]
        InspectAccount["계정 연결 상태 확인"]

        Operator --> OperatorLogin
        Operator --> IssueTestToken
        Operator --> ManageRoles
        Operator --> InspectAccount
    end

    subgraph ServiceArea["대상 서비스"]
        direction TB
        Auth["인증 도메인"]
        User["사용자 도메인"]
    end

    CustomerArea --> Auth
    CustomerArea --> User
    SellerArea --> Auth
    SellerArea --> User
    OperatorArea --> Auth
    OperatorArea --> User
```

### 2.5 RBAC 요구사항 매트릭스

| 기능 | anonymous | CUSTOMER | PROVIDER | ADMIN |
| --- | --- | --- | --- | --- |
| 공개 상품/드롭 조회 | 가능 | 가능 | 가능 | 가능 |
| 이메일/비밀번호 로그인 | 불가 | 가능 | 가능 | 가능 |
| OAuth/OIDC 로그인 | 불가 | 가능 | 가능 | 가능 |
| 내 정보 조회 | 불가 | 본인만 가능 | 본인만 가능 | 가능 |
| 인증 수단 연결 | 불가 | 본인만 가능 | 본인만 가능 | 가능 |
| 비밀번호 재설정 | 불가 | 불가 | 불가 | 불가 |
| 주문 생성 | 불가 | 가능 | 정책 결정 필요 | 불가 |
| 판매 상품/드롭 관리 | 불가 | 불가 | 가능 | 가능 |
| 계정 연결 상태 확인 | 불가 | 본인만 가능 | 본인만 가능 | 가능 |
| 사용자/판매자 권한 관리 | 불가 | 불가 | 불가 | 가능 |
| 계정 비활성화 | 불가 | 불가 | 불가 | 불가 |
| 테스트 계정 토큰 발급 | 불가 | 불가 | 불가 | 가능 |

기본 인가 판단은 role을 우선 사용한다. 단, 특정 사용자, 특정 리소스, 특정 기능에 대한 예외 허용 또는 예외 차단이 필요하면 ACL override를 적용할 수 있어야 한다.

ACL override 예시:

- 특정 판매자에게 특정 드롭 관리 권한만 허용
- 특정 운영자에게 계정 조회는 허용하지만 권한 변경은 차단
- 특정 사용자에게 테스트 또는 베타 기능 접근 허용
- 비정상 계정 또는 제재 계정의 일부 기능 차단
- QA 계정에 운영 환경과 분리된 테스트 권한 부여

### 2.6 인가 판단 순서

```text
authentication result
-> principal normalization
-> user status check
-> RBAC role policy
-> ownership check
-> ACL override allow/deny
-> authorization decision
```

ACL override는 role 정책을 보완하기 위한 예외 규칙이다. 모든 권한을 ACL로만 관리하지 않는다.

### 2.7 기능적 요구사항

#### FR-001 OAuth 동일 이메일 계정 처리

OAuth 또는 OIDC 최초 로그인 시 제공자에서 받은 이메일과 같은 이메일을 가진 기존 인증 계정 또는 사용자 계정이 있으면 자동 연결하지 않는다.

사용자에게 기존 계정에 인증 수단을 연동할지, 새 계정을 만들지 선택할 수 있게 한다. 기존 계정 연동을 선택한 경우에는 기존 계정에 대한 추가 인증을 요구한다.

#### FR-002 OAuth/OIDC 고유 식별자 기준

서로 다른 OAuth/OIDC 제공자가 같은 이메일을 반환하면 같은 사용자 후보로 볼 수 있다. 단, 이메일은 보조 식별자일 뿐이며 인증 계정의 고유 식별자로 사용하지 않는다.

OAuth/OIDC 인증 계정은 제공자와 제공자가 발급한 원본 subject 조합으로 식별한다.

```text
auth_provider + provider_subject
```

계정 연결 후 로그인 식별도 이메일이 아니라 제공자와 원본 subject 조합을 기준으로 한다.

#### FR-003 인증 수단 연결 개수 제한

한 사용자 계정에 연결 가능한 인증 수단 수는 별도로 제한하지 않는다.

지원하는 인증 수단과 제공자 수가 정해져 있으므로 실제 연결 가능 수는 시스템이 지원하는 인증 수단 범위 안에서 자연스럽게 제한된다.

#### FR-004 비밀번호 찾기 제외

MVP에서는 이메일 인증을 이용한 비밀번호 찾기 또는 비밀번호 재설정 기능을 제공하지 않는다.

비밀번호 찾기는 이메일 발송, 재설정 토큰 만료, 토큰 재사용 방지, 계정 열거 방지, 감사 로그가 필요하므로 이후 범위에서 별도로 설계한다.

#### FR-005 로그아웃 범위

기본 로그아웃은 현재 세션만 종료한다.

모든 기기 세션 종료는 MVP 필수 기능으로 포함하지 않는다. 이후 보안 기능으로 별도 설계할 수 있다.

#### FR-006 비로그인 공개 기능 범위

비로그인 사용자가 접근 가능한 공개 기능 범위는 인증/사용자 시나리오에서 고정 목록으로 확정하지 않는다.

공개 접근 여부는 각 피처가 route 또는 action 단위로 정의한다. 인증/사용자 서비스는 인증 정보가 없는 요청을 anonymous Principal로 표현하고, 각 피처의 인가 정책이 public 접근 가능 여부를 판단할 수 있게 한다.

#### FR-007 내부 서비스 사용자 식별 헤더 전달

Istio IngressGateway는 외부 인증 정보를 검증한 뒤 내부 서비스가 사용할 수 있는 사용자 식별 헤더를 생성한다.

표준 전달 헤더는 다음을 기본으로 한다.

```http
X-User-Id: <jwt.sub>
X-User-Role: <jwt.role>
X-Token-Id: <jwt.jti>
```

`X-User-Id`, `X-User-Role`, `X-Token-Id`는 보호 API의 필수 헤더다. 이메일은 개인정보이므로 내부 전달 헤더로 노출하지 않는다. 이메일은 auth-service 내부의 credential 식별과 감사 기록에 필요한 범위에서만 다룬다.

내부 서비스는 원본 JWT를 직접 디코딩하지 않는다. 인증/인가가 필요한 API는 Istio IngressGateway가 검증 후 생성한 `X-User-*` 헤더를 애플리케이션 레벨의 Principal 또는 AuthContext로 변환해 인가 판단을 수행한다.

내부 서비스로 전달되는 `X-User-*`는 Istio IngressGateway가 검증된 JWT claim으로 생성한 값이어야 한다.

근거:

- [OWASP Testing JSON Web Tokens](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/06-Session_Management_Testing/10-Testing_JSON_Web_Tokens)는 JWT가 인증/세션 토큰으로 쓰일 때 변조, 민감정보 노출, 검증 오류가 계정 탈취로 이어질 수 있음을 테스트 관점에서 다룬다. 따라서 원본 JWT 검증 책임은 Istio IngressGateway 같은 접근 경계에 모으고, 내부 서비스에는 검증 완료된 사용자 식별 헤더만 전달한다.
- [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)는 JWT 사용 시 토큰 저장, 폐기, 정보 노출, 서명 검증 같은 보안 고려사항을 별도로 다룬다. 내부 서비스가 각자 JWT claim 구조와 검증 방식을 중복 구현하지 않도록 접근 경계에서 identity header 변환 계층을 둔다.
- [RFC 8252 OAuth 2.0 for Native Apps](https://datatracker.ietf.org/doc/html/rfc8252)는 native/mobile 앱 인증에서 앱과 인증 처리 경계를 분리하고 외부 user-agent 사용을 권장한다. 클라이언트 인증 입력을 내부 서비스 모델과 분리해야 한다는 설계 근거로 사용한다.

#### FR-008 ACL Override 평가 규칙

ACL override는 allow와 deny를 모두 지원한다.

기본 정책은 deny by default로 둔다. RBAC 또는 ACL allow가 있어야 허용 후보가 되며, 적용 가능한 ACL deny가 하나라도 있으면 최종 거부한다.

allow와 deny가 동시에 매칭되면 deny가 우선한다. 넓은 권한을 부여한 뒤 특정 기능이나 리소스를 명시적으로 차단하는 운영 방식을 지원한다.

특정 권한 검증은 role, ACL override, 세션 상태 같은 최신 정책 상태를 함께 봐야 하므로 stateful하다. 이 경우 도메인 서비스는 auth-service에 권한 정책 확인을 요청한다. Redis는 auth-service가 정책 상태 조회와 무효화에 사용하는 필수 구성요소다. ([NFR-007](#nfr-007-권한-정책-조회))

#### FR-009 공통 사용자 식별 헤더 정규화

웹과 모바일의 access token은 Istio IngressGateway에서 검증한 뒤 공통 `X-User-*` 헤더로 정규화한다.

인증/인가가 필요한 API는 클라이언트 종류에 의존하지 않고, 정규화된 `X-User-*` 헤더를 서비스 내부 Principal 또는 AuthContext로 변환해 인가 판단을 수행한다.

#### FR-010 사용자 탈퇴와 계정 비활성화 제외

MVP에서는 사용자 탈퇴와 계정 비활성화 기능을 제공하지 않는다.

MVP 범위는 인증 계정 생성, 사용자 계정 생성, 로그인, 인증 수단 연결에 집중한다.

#### FR-011 인증 수단 연결 해제 제외

MVP에서는 사용자가 연결된 인증 수단을 해제하는 기능을 제공하지 않는다.

인증 수단 연결 해제는 마지막 인증 수단 보호, 재인증, audit log가 필요하므로 이후 범위에서 별도로 설계한다.

#### FR-012 가입 시 동의 정보 제외

MVP에서는 가입 시점에 약관, 개인정보, 마케팅, 알림 동의 정보를 받지 않는다.

동의 정보는 구매, 알림 설정, 마케팅 기능이 구체화되는 시점에 별도 요구사항으로 다룬다.

#### FR-013 이메일 가입 최소 입력 정보

MVP의 이메일 사용자 가입은 인증 계정 생성에 필요한 최소 입력 정보와 사용자 프로필 초기화 정보를 분리해서 받는다.

auth-service가 처리하는 필수 입력 정보:

- 이메일
- 비밀번호

user-service가 처리하는 프로필 초기화 정보:

- 실명

클라이언트는 이를 하나의 회원가입 화면에서 받을 수 있지만, 서비스 경계에서는 이메일/비밀번호는 auth-service로, 실명 같은 사용자 프로필은 user-service로 전달한다.

#### FR-014 이메일 소유 검증 목킹

MVP에서는 이메일 소유 검증 단계가 있다고 가정하되, 실제 이메일 발송과 인증 링크 처리는 상세 설계하지 않는다.

개발과 테스트에서는 이메일 검증을 목킹할 수 있어야 한다.

### 2.8 비기능적 요구사항

#### NFR-001 사용자 생성의 최종적 일관성

이메일 사용자 가입은 auth-service가 이메일/비밀번호 `AuthAccount`와 내부 `user_id`를 먼저 생성하는 방식으로 처리한다. user-service는 해당 `user_id`로 프로필 초기화 요청을 받거나, 최초 접근 시 사용자 정보가 없으면 기본 사용자 정보를 생성한다.

이 방식은 인증 계정 생성과 사용자 프로필 생성을 하나의 분산 트랜잭션으로 묶지 않는다. 사용자 서비스는 "없어서 만들었다" 방식으로 사용자 정보를 지연 생성할 수 있어야 한다.

#### NFR-002 인증과 사용자 서비스의 독립 배포

인증 서비스와 사용자 서비스는 서로 다른 배포 단위로 동작할 수 있어야 한다. 인증 서비스의 사용자 ID 발급이 성공했더라도 사용자 서비스의 프로필 생성은 지연될 수 있다.

#### NFR-003 사용자 지연 생성의 멱등성

같은 사용자 ID로 사용자 정보 생성 요청이 반복되어도 사용자 정보는 하나만 생성되어야 한다. 중복 요청은 기존 사용자 정보를 반환해야 한다.

#### NFR-004 테스트 계정 토큰 발급 제약

테스트 계정 토큰 발급은 개발 빌드에서만 인증 서비스 API로 제공한다.

별도 생성 도구는 제공하지 않는다. 호출자가 전달한 고유한 토큰 문자열을 기반으로 테스트 계정을 발급할 수 있어야 한다.

#### NFR-005 내부 서비스의 인증 토큰 포맷 독립성

내부 서비스는 JWT claim 구조나 토큰 포맷 변경에 직접 의존하지 않아야 한다.

JWT 검증 방식이 변경되더라도 Istio IngressGateway의 `X-User-*` 변환 계층만 변경해서 내부 서비스를 무중단 배포할 수 있어야 한다.

#### NFR-006 고위험 서비스의 인증 계약 버전 전환

고위험 서비스는 인증 시크릿, JWT 서명 키, 사용자 식별 헤더 구조가 변경될 때 새 인증 계약 버전으로 전환할 수 있어야 한다.

예를 들어 신규 JWT 인증 시크릿으로 변경되는 경우 Istio IngressGateway는 새 버전의 `X-User-*` 헤더 계약을 만들고, 고위험 서비스는 해당 버전을 지원하는 v2 배포로 전환할 수 있어야 한다.

기존 토큰은 만료 또는 재로그인을 통해 새 토큰으로 교체되도록 유도한다.

#### NFR-007 권한 정책 조회

특정 권한에 대한 검증은 stateful하게 처리한다.

role grant, ACL override, 세션 폐기 상태, policy version, session version은 최신 정책 상태를 봐야 하므로 매 요청마다 저장소를 직접 조회하면 auth-service 저장소가 병목이 된다.

권한 검증이 필요한 도메인 서비스는 auth-service에 user_id, role, token_id, resource, action을 전달해 권한 정책을 확인한다.

auth-service는 Redis를 정책 상태 저장소로 사용한다. role 변경, ACL override 변경, 로그아웃, 세션 폐기 시 관련 정책 상태를 무효화할 수 있어야 한다.

ACL override 변경 API는 변경 이력 기록과 정책 상태 무효화를 함께 수행해야 한다. 무효화에 실패하면 ACL 변경도 성공으로 응답하지 않는다.

일반 요청은 Redis miss 시 저장소 조회로 보강할 수 있지만, 고위험 요청은 최신 정책 확인에 실패하면 거부하는 쪽을 기본으로 둔다.

#### NFR-008 토큰 검증 강도 분리

모든 요청에 같은 수준의 토큰 검증과 권한 검증을 적용하지 않는다.

일반 사용자 API는 대부분 access token의 stateless 검증과 `X-User-*` 기반 판단으로 처리한다. auth-service 장애가 발생해도 유효한 access token이 있으면 Istio IngressGateway가 검증을 수행하고 기존 사용자 요청을 일정 시간 계속 처리할 수 있어야 한다.

운영, 보안, 권한 변경, 세션 폐기, ACL override 변경, 테스트 토큰 발급 같은 고위험 API는 최신 세션 상태와 최신 권한 정책을 확인하는 stateful 검증을 적용한다.

고위험 API는 최신 정책 확인에 실패하면 거부한다. 이 요청들은 장애 상황에서 가용성보다 안전성을 우선한다.

## 3. 정상 시나리오

### 3.1 이메일 회원가입

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant AUTH as 인증 도메인
    participant USER as 사용자 도메인

    U->>FE: 이메일, 비밀번호, 프로필 정보 입력
    FE->>GW: 이메일 사용 가능 여부 확인
    GW->>AUTH: 이메일 credential 중복 확인
    AUTH-->>GW: 사용 가능
    GW-->>FE: 사용 가능
    FE->>GW: 이메일 소유 검증 시작
    GW->>AUTH: 검증 코드 또는 링크 발급
    AUTH-->>GW: 검증 요청 생성
    GW-->>FE: 검증 안내
    U->>FE: 이메일 검증 완료
    FE->>GW: 회원가입 완료 요청
    GW->>AUTH: 이메일/비밀번호 AuthAccount 생성
    AUTH->>AUTH: user_id 발급
    AUTH->>AUTH: AuthAccount와 user_id 연결
    AUTH-->>GW: user_id 반환
    GW->>USER: user_id 기반 사용자 프로필 초기화
    USER->>USER: UserStatus=active 저장
    USER-->>GW: 사용자 생성 완료
    GW->>AUTH: 세션 및 토큰 발급
    AUTH-->>GW: access token, refresh token
    GW-->>FE: 가입 완료 및 로그인 상태 반환
```

일반적인 웹/모바일 회원가입에서는 이메일 소유 검증을 먼저 끝낸 뒤 계정을 활성화한다. MVP에서는 실제 이메일 발송을 목킹할 수 있지만, 시퀀스상으로는 검증 단계를 별도 단계로 둔다.

프로필 초기화가 실패하면 auth-service의 `AuthAccount`와 `user_id`는 이미 생성될 수 있다. 이 경우 user-service는 같은 `user_id`에 대한 프로필 초기화 재시도를 멱등하게 처리해야 한다.

### 3.2 이메일 로그인

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant AUTH as 인증 도메인

    U->>FE: 이메일/비밀번호 로그인
    FE->>GW: 로그인 요청
    GW->>AUTH: 인증 정보 검증
    AUTH-->>GW: 인증 완료
    GW->>GW: X-User-* 헤더 생성
    GW-->>FE: 로그인 완료
```

### 3.3 OAuth 최초 로그인 후 사용자 생성

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant OP as OAuth/OIDC provider
    participant AUTH as 인증 도메인
    participant USER as 사용자 도메인

    U->>FE: OAuth/OIDC 로그인 선택
    FE->>OP: provider 인증 요청
    OP-->>FE: 인증 결과 반환
    FE->>GW: provider 인증 결과 전달
    GW->>AUTH: provider + subject 검증
    AUTH->>AUTH: OAuth/OIDC AuthAccount 생성 또는 조회
    AUTH->>AUTH: AuthAccount와 user_id 연결
    AUTH-->>GW: user_id 발급 또는 조회
    GW->>USER: user_id 기반 사용자 정보 생성 또는 조회
    USER-->>GW: 사용자 정보 반환
    GW->>GW: X-User-* 헤더 생성
    GW-->>FE: 로그인 완료
```

### 3.4 비로그인 공개 기능 접근

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant SVC as feature-service

    U->>FE: 공개 기능 접근
    FE->>GW: 인증 정보 없는 요청
    GW->>GW: 인증 헤더 없음 확인
    GW->>SVC: X-User-* 없이 공개 요청 전달
    SVC->>SVC: public route/action 정책 확인
    SVC-->>GW: 공개 응답
    GW-->>FE: 공개 응답
```

### 3.5 기존 사용자의 인증 수단 추가 연결

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant OP as OAuth/OIDC provider
    participant AUTH as 인증 도메인

    U->>FE: 새 인증 수단 연결 선택
    FE->>GW: 현재 access token 기반 연결 시작
    GW->>AUTH: X-User-Id 기반 현재 사용자 확인
    AUTH-->>GW: 연결 가능 상태 반환
    FE->>OP: 새 provider 인증 요청
    OP-->>FE: provider 인증 결과 반환
    FE->>GW: provider 인증 결과 전달
    GW->>AUTH: provider + subject로 새 AuthAccount 연결 요청
    AUTH->>AUTH: 새 AuthAccount를 현재 user_id에 연결
    AUTH-->>GW: 인증 수단 연결 완료
    GW-->>FE: 연결 완료
```

### 3.6 여러 인증 수단으로 같은 사용자 계정 접속

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant AUTH as 인증 도메인

    U->>FE: 연결된 인증 수단으로 로그인
    FE->>GW: 로그인 요청
    GW->>AUTH: 인증 수단별 AuthAccount 조회
    AUTH-->>GW: 같은 user_id 반환
    GW->>GW: 같은 user_id로 X-User-* 헤더 생성
    GW-->>FE: 로그인 완료
```

### 3.7 인증 유지와 토큰 갱신

```mermaid
sequenceDiagram
    autonumber
    participant C as 웹/앱
    participant GW as Istio IngressGateway
    participant AUTH as 인증 도메인

    C->>GW: refresh token으로 갱신 요청
    GW->>AUTH: refresh token 갱신 요청
    AUTH-->>GW: 갱신된 인증 상태 반환
    GW->>GW: X-User-* 헤더 재생성
    GW-->>C: 갱신 완료
```

### 3.8 내 정보 확인

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant USER as 사용자 도메인

    U->>FE: 내 정보 확인
    FE->>GW: 내 정보 요청
    GW->>GW: X-User-Id 확인
    GW->>USER: user_id 기반 내 정보 조회
    USER-->>GW: 사용자 정보 반환
    GW-->>FE: 내 정보 표시
```

## 4. 예외 시나리오

### 4.1 이메일 또는 비밀번호 불일치

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant AUTH as 인증 도메인

    U->>FE: 이메일/비밀번호 로그인
    FE->>GW: 로그인 요청
    GW->>AUTH: 인증 정보 검증
    AUTH-->>GW: 인증 실패
    GW-->>FE: 로그인 실패 응답
    FE-->>U: 실패 안내
```

### 4.2 이미 연결된 인증 수단

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant AUTH as 인증 도메인

    U->>FE: 인증 수단 연결 요청
    FE->>GW: provider 인증 결과 전달
    GW->>AUTH: provider + subject 연결 요청
    AUTH-->>GW: 이미 연결된 인증 수단
    GW-->>FE: 연결 불가 응답
    FE-->>U: 이미 연결됨 안내
```

### 4.3 다른 사용자에 연결된 인증 수단

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant AUTH as 인증 도메인

    U->>FE: 인증 수단 연결 요청
    FE->>GW: provider 인증 결과 전달
    GW->>AUTH: provider + subject 연결 요청
    AUTH-->>GW: 다른 user_id에 연결됨
    GW-->>FE: 연결 충돌 응답
    FE-->>U: 다른 계정에 연결된 수단 안내
```

### 4.4 OAuth 제공자 응답 불일치

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant OP as OAuth/OIDC provider
    participant AUTH as 인증 도메인

    U->>FE: OAuth/OIDC 로그인
    FE->>OP: provider 인증 요청
    OP-->>FE: 불완전하거나 검증 불가한 응답
    FE->>GW: provider 인증 결과 전달
    GW->>AUTH: provider 응답 검증
    AUTH-->>GW: provider 응답 거부
    GW-->>FE: 로그인 실패 응답
```

### 4.5 인증 계정은 존재하지만 사용자 생성이 지연된 상태

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant AUTH as 인증 도메인
    participant USER as 사용자 도메인

    U->>FE: 로그인 후 사용자 기능 접근
    FE->>GW: 사용자 기능 요청
    GW->>AUTH: 인증 상태 확인
    AUTH-->>GW: user_id 반환
    GW->>USER: user_id 기반 사용자 정보 조회
    USER->>USER: 사용자 정보 없으면 생성
    USER-->>GW: 사용자 정보 반환
    GW-->>FE: 사용자 기능 응답
```

### 4.6 비로그인 사용자가 보호 기능에 접근한 상태

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 웹/앱
    participant GW as Istio IngressGateway
    participant SVC as feature-service

    U->>FE: 보호 기능 접근
    FE->>GW: 인증 정보 없는 요청
    GW->>GW: 인증 헤더 없음 확인
    GW->>SVC: X-User-* 없이 보호 요청 전달
    SVC->>SVC: protected route/action 정책 확인
    SVC-->>GW: 접근 거부
    GW-->>FE: 로그인 필요 응답
```

## 5. 완료 기준

- 인증 계정과 사용자 계정의 차이가 사용자 시나리오에서 드러난다.
- 하나의 사용자이 여러 인증 수단으로 로그인하는 흐름이 정의된다.
- 비로그인 사용자가 접근 가능한 공개 기능과 보호 기능의 경계가 정의된다.
- 테스트 계정 토큰 발급이 실사용자 인증 수단과 분리된다.
- 계정 연결, 중복 연결, 충돌 상황의 사용자 처리가 정의된다.
- 이후 컨텍스트 바운더리와 서비스 설계에서 참조할 수 있는 정상/예외 시나리오가 정리된다.
