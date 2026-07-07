---
id: SCN.A.XX
title: 시나리오 이름
type: scenario
status: draft
tags: [scenario, mermaid, ui, backend]
source: local
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# 시나리오 이름

## 기본 정보

- Scenario ID: `SCN.A.XX`
- 시작 지점:
- 트리거:
- 성공 기준:
- 실패 기준:

## 연관 태그

🏷️ 플로우 참조: FLOW.A.XX | UC 참조: [UC.A.XX](../30-uc/UC_A_XX_name.md) | 영속성 참조: [PST.A.XX](../55-persistence/PST_A_XX_name.md) | 서비스 참조: [SVC.A.XX](../60-service/SVC_A_XX_name.md) | API 참조: [API.A.XX](../70-api/API_A_XX_name.md) | UI 참조: [UI.A.XX](../20-ui/UI_A_XX_name.md) | 페이지 참조: [PAGE.A.XX](../10-sitemap/PAGE_A_XX_name.md) | 도메인 참조: [AGG.A.XX](../50-domain-model/AGG_A_XX_name.md)

## 처리 과정

```mermaid
sequenceDiagram
    actor User
    participant Page as PAGE.A.XX
    participant API as API.A.XX
    participant App as Application Service
    participant Agg as AGG-EXAMPLE
    participant DB as Database

    User->>Page: 사용자 행동
    Page->>API: 요청
    API->>App: Command 또는 Query 위임
    App->>Agg: 도메인 모델 실행
    Agg-->>App: 결과
    App->>DB: 저장 또는 조회
    DB-->>App: 결과
    App-->>API: 응답 모델
    API-->>Page: 응답
    Page-->>User: 피드백 표시
```

## 단계 설명

| 단계 | 주체 | 설명 | 관련 식별자 |
| --- | --- | --- | --- |
| 1 | User |  |  |
| 2 | Page |  |  |
| 3 | API |  |  |

## 데이터 이동

- 입력:
- 출력:
- 저장:
- 발행 Event:

## 예외 흐름

-

## 확인 필요

-
