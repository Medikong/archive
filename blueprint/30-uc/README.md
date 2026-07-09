---
id: use-case-context
title: 유스케이스 작성 컨텍스트
type: use-case-context
status: draft
tags: [use-case, context, guide, dropmong]
source: local
created: 2026-07-06
updated: 2026-07-08
---

# 유스케이스 작성 컨텍스트

## 기준 문서

- [유스케이스 인덱스](INDEX.md)
- [유스케이스 템플릿](.template/use-case.md)

Use Case Diagram을 작성하세요.

### 작성 규칙

* **Actor**

  * 시스템 외부의 사용자 또는 외부 시스템
  * 시스템 바운더리 밖에 배치
  * 역할(Role)을 나타내는 **명사**를 사용

    * 예) 고객, 운영자, 판매자, 결제 시스템

* **Use Case**

  * 원형 노드(Use Case)로 표현
  * **명사 또는 명사구** 형태로 작성

    * 예) 로그인, 상품 조회, 주문 생성, 결제 요청
  * 시스템이 제공하는 기능만 포함

* **System Boundary**

  * 시스템 전체를 하나의 Boundary로 감싼다.
  * Boundary 이름은 시스템명과 같은 **명사**를 사용

* **Association**

  * Actor와 Use Case를 연결한다.

* **include**

  * 항상 수행되는 공통 기능
  * 여러 Use Case에서 재사용되는 기능

* **extend**

  * 조건부 또는 선택적으로 수행되는 기능
  * 기존 Use Case를 확장하는 기능

* **Generalization**

  * Actor 또는 Use Case의 상속 관계를 표현한다.

### 작성 원칙

* Actor, System Boundary, Use Case는 **명사**를 사용한다.
* UI 화면이나 버튼명이 아닌 사용자의 목표(User Goal)를 중심으로 작성한다.
* 기술 구현이 아닌 비즈니스 기능을 표현한다.
* 구매자 유스케이스는 인증이 완료된 사용자를 전제로 하며, 모든 구매자 유스케이스에 인증을 반복해서 include하지 않는다.
* 기능이 많으면 여러 개의 Use Case Diagram으로 분리한다.
