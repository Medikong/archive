# Toss Cert API

- 회사: Toss
- 원문: https://developers-apps-in-toss.toss.im/tossauth/develop.html
- 발행일: 미표기
- 접근일: 2026-07-04
- 주제: 본인확인 API, txId, sessionKey, 인증 상태, 서버 간 결과 조회

## 원문 확인 내용

[토스 인증 API 문서](https://developers-apps-in-toss.toss.im/tossauth/develop.html)는 인증 요청 시 토스 인증 서버가 `txId`를 발급하고, 사용자가 앱에서 인증을 진행하는 구조를 설명한다. 상태 값은 요청됨, 진행 중, 완료, 만료처럼 transaction 중심으로 표현된다.

[토스 인증 API 문서](https://developers-apps-in-toss.toss.im/tossauth/develop.html)는 결과 조회를 서버 간 통신으로 진행하라고 안내한다. 인증이 완료된 뒤에도 결과 조회 API로 최종 판단해야 하며, 성공 기준 조회 횟수와 조회 가능 시간에 제한이 있다.

[토스 인증 API 문서](https://developers-apps-in-toss.toss.im/tossauth/develop.html)는 개인정보 기반 인증과 결과 조회에서 `sessionKey`를 매 요청 새로 생성해야 한다고 안내한다.

## 적용 아이디어

- DropMong의 민감 작업 인증은 `auth_challenge` transaction으로 모델링한다.
- 프론트엔드 완료 신호만 믿지 않고, 서버가 challenge 결과를 조회하거나 검증한 뒤 상태를 확정한다.
- challenge 결과 조회 횟수와 유효 시간을 제한하고, 실패/만료는 감사 로그와 rate limit 입력으로 사용한다.

## 확인 필요

- [토스 인증 API 문서](https://developers-apps-in-toss.toss.im/tossauth/develop.html)는 외부 고객사가 사용하는 본인확인 API 문서다. Toss 내부 계정 모델을 직접 설명하지는 않는다.
