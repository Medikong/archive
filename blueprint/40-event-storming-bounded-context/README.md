---
id: bounded-context-index
title: 이벤트스토밍과 바운디드 컨텍스트 인덱스
type: bounded-context-index
status: example
tags: [event-storming, bounded-context, index, example]
source: local
created: 2026-07-06
updated: 2026-07-06
---

# 이벤트스토밍과 바운디드 컨텍스트 인덱스

## 역할

유스케이스에서 드러난 도메인 이벤트, 용어, 정책, 컨텍스트 경계를 정리한다.

## 템플릿

- [이벤트스토밍과 바운디드 컨텍스트 템플릿](.template/bounded-context.md)

## 디자인 가이드

- [이벤트스토밍 다이어그램 디자인 가이드](DESIGN.md)
- 다이어그램을 작성하거나 수정하기 전에는 [DESIGN.md](DESIGN.md)를 먼저 읽고, 이 문서에는 결과 해석에 필요한 보조 표와 설계 판단만 남긴다.

## 예시 문서

- [BC.A.01 주문 이벤트스토밍과 바운디드 컨텍스트](.examples/BC_A_01_order.md)

## 작성 절차와 문서 구성

이벤트스토밍은 설계 과정에서는 시간 순서로 도출하지만, 문서에서는 최종 설계 결과를 먼저 보여준다. 문서 독자는 설계 과정을 따라가는 사람이 아니라, 현재 시스템 경계와 이벤트 관계를 빠르게 이해해야 하는 사람이다.

### 설계 작업 순서

작업자는 다음 순서로 요소를 도출한다.

1. `Domain Event`를 먼저 찾는다. 이벤트는 이미 발생한 결과이므로 과거형으로 적는다.
2. 이벤트를 발생시키는 `Command`와 실행 주체인 `Actor`를 찾는다.
3. 커맨드와 이벤트가 다루는 `Aggregate`를 배치한다.
4. 기획상 또는 개발자 관점에서 제한해야 하는 `Policy`를 찾는다.
5. 서비스 지향점에서 도출한 `Business Rule`을 찾는다.
6. 아직 결정이 필요한 `Hotspot`을 표시한다.
7. 관련 요소를 `Bounded Context`로 묶고, 컨텍스트 간 외부 시스템 관계를 정리한다.

이 순서는 설계를 만들기 위한 내부 절차다. 실제 문서에 이 순서를 그대로 노출하지 않는다.

### 최종 문서 구성

문서는 결과 중심으로 구성한다.

1. 기본 정보와 연관 태그
2. 컨텍스트 경계 요약
3. Event Storming Diagram
4. Element Catalog
5. Element Evidence
6. Event Relations
7. 유비쿼터스 언어
8. 후속 설계 메모

## 다이어그램 작성 규칙

Mermaid `flowchart`를 사용하여 최종 이벤트 관계와 컨텍스트 경계를 표현한다. 레이아웃, 노드 도형, 서브그래프 표현, 관계선, 라이트/다크 테마 색상은 [DESIGN.md](DESIGN.md)를 따른다.

다이어그램을 작성하거나 수정하기 전에는 [DESIGN.md](DESIGN.md)를 먼저 읽고, 이 문서에는 결과 해석에 필요한 보조 표와 설계 판단만 남긴다.

## Element Catalog

다이어그램 아래에는 Markdown Table로 보조 자료를 제공한다. 표는 다이어그램을 설명하는 보조 자료이며, 다이어그램에 넣기 어려운 설명과 참조를 담는다.

### 작성 원칙

- `Element Catalog`는 요소 나열만 담당한다.
- `Element Evidence`는 사전 문서에서 각 요소를 도출한 근거만 담당한다.
- 요소 간 연결은 `Event Relations`에서만 다룬다.
- 다이어그램은 최종 설계 결과를 보여주고, 설계 도출 과정은 보여주지 않는다.
- 노드에 긴 설명을 넣지 않고, 설명은 표로 보낸다.
- 소속 컨텍스트가 없거나 외부에 있는 요소는 `Context 외부`로 적는다.

| 유형 | 식별자 | 이름 | 소속 컨텍스트 | 설명 |
| --- | --- | --- | --- | --- |
| Actor | ACTOR.A.XX |  | Context 외부 | 커맨드를 발생시키는 주체 |
| Command | CMD.A.XX |  |  | 액터나 정책이 요청하는 행위 |
| Aggregate | AGG.A.XX |  |  | 커맨드와 이벤트가 다루는 핵심 데이터 |
| Domain Event | EVT.A.XX |  |  | 도메인에서 이미 발생한 결과 |
| Policy | POLICY.A.XX |  |  | 기획상 또는 개발자 관점에서 제한해야 하는 제약 |
| Business Rule | RULE.A.XX |  |  | 서비스 지향점에서 도출한 도메인 규칙 |
| Hotspot | HOTSPOT.A.XX |  |  | 아직 결정이 필요한 논의 지점 |
| External System | EXT.A.XX |  | Context 외부 | 외부 의존 시스템 |
| Read Model | RM.A.XX |  |  | 결과 확인을 위한 조회 모델 |

## Element Evidence

`REQ`, `PAGE`, `UI`, `UC` 등 사전 문서를 분석해 각 요소를 도출한 근거를 적는다. 이 표는 관계를 설명하지 않고, 요소가 왜 필요한지에 대한 추적 근거만 남긴다.

| 요소 | 근거 문서 | 근거 내용 |
| --- | --- | --- |
| ACTOR.A.XX | UC.A.XX |  |
| CMD.A.XX | UC.A.XX, PAGE.A.XX, UI.A.XX |  |
| AGG.A.XX | REQ.A.XX, UC.A.XX |  |
| EVT.A.XX | UC.A.XX |  |
| POLICY.A.XX | REQ.A.XX, UC.A.XX |  |
| RULE.A.XX | REQ.A.XX, UC.A.XX |  |
| HOTSPOT.A.XX | REQ.A.XX, PAGE.A.XX, UI.A.XX, UC.A.XX |  |
| EXT.A.XX | REQ.A.XX, UC.A.XX |  |
| RM.A.XX | PAGE.A.XX, UI.A.XX |  |

## 후속 설계 메모

도메인 모델, 영속성, 서비스, API 상세는 각 전용 문서에서 다룬다. 이 문서에는 후속 설계에서 놓치면 안 되는 연결 메모만 남긴다.

| 항목 | 메모 | 연결 문서 |
| --- | --- | --- |
| 도메인 모델 | Aggregate, Entity, Value Object로 넘길 내용 | `AGG.A.XX` |
| 영속성 | 저장소, 트랜잭션, 인덱스, outbox로 넘길 내용 | `PST.A.XX` |
| 서비스 | 유스케이스 서비스, 정책 실행 위치로 넘길 내용 | `SVC.A.XX` |
| API | 요청/응답, 인증, 에러로 넘길 내용 | `API.A.XX` |
| 발행 Event | 다른 BC에 알리는 이벤트 |  |
| 구독 Event | 다른 BC에서 받아 반응하는 이벤트 |  |
| 외부 연동 | 외부 시스템 의존성 |  |
| 정책/불변조건 | 다이어그램에 넣기 어려운 핵심 규칙 |  |
| 열린 질문 | 아직 결정하지 않은 경계와 책임 |  |
