---
title: 인증과 사용자 API 설계
status: draft
created: 2026-07-03
updated: 2026-07-03
owner: TBD
scope: auth-user
---

# 인증과 사용자 API 설계

## 1. 목적

인증과 사용자 흐름을 외부 API, 내부 API, 개발 빌드 전용 API로 나누어 정의한다.

## 검토 질문

1. 이메일 사용자 가입 API는 인증 정보와 사용자 정보를 한 요청에서 받을까요?
2. OAuth/OIDC 로그인 API는 provider별 callback을 분리할까요, 공통 endpoint로 받을까요?
3. 동일 이메일 후보가 있을 때 계정 연결 또는 새 계정 생성을 선택하는 API는 어떻게 나눌까요?
4. 인증 수단 연결 API는 재인증 또는 step-up 인증을 요구하나요?
5. 로그아웃 API는 현재 세션만 종료하는 계약으로 둘까요?
6. 내 정보 조회 API는 사용자 도메인 API로만 제공하나요?
7. 개발 빌드 전용 테스트 토큰 발급 API는 어떤 route prefix로 격리하나요?
8. 비밀번호 찾기/재설정 API는 MVP 계약에서 완전히 제외하나요?
9. gateway/BFF가 내부 서비스에 전달하는 Principal payload는 HTTP header, body, 또는 내부 request context 중 무엇으로 전달하나요?
10. 외부 API 에러 코드는 인증 실패, 인가 실패, 계정 연결 충돌, 사용자 비활성 상태를 어떻게 구분하나요?
