---
id: SCN.A.01
title: 사용자 목적 시퀀스 인덱스
type: sequence-index
status: draft
tags: [sequence, scenario, user]
source: local
created: 2026-07-15
updated: 2026-07-15
---

# 사용자 목적 시퀀스 인덱스

여러 Context가 함께 참여하는 사용자 목적의 처리 순서를 관리한다. 각 서비스 내부의 Handler, 트랜잭션과 저장 규칙은 서비스 디자인 문서가 담당한다.

| ID | 처리 | 참여 Context | 문서 |
| --- | --- | --- | --- |
| `SCN.A.01-01` | 회원가입과 자동 로그인 | 프론트엔드, Ingress, Auth, User, 이메일·SMS Provider | [상세](SCN_A_01_01_user_provisioning_auth_link.md) |
