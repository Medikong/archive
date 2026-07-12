---
id: sequence-index
title: 처리 시퀀스 인덱스
type: sequence-index
status: draft
tags: [scenario, sequence, index]
source: local
created: 2026-07-06
updated: 2026-07-10
---

# 처리 시퀀스 인덱스

## 역할

특정 화면 행동, 여러 API 호출, 백그라운드 작업과 Context 간 상호작용을 시나리오 단위의 시퀀스로 정리한다. 논리 식별자는 기존 추적성을 유지하기 위해 `SCN`을 사용한다.

## 템플릿

- [처리 시퀀스 템플릿](.template/scenario.md)

## 실제 문서

- [SCN.A.300 인증 처리 시퀀스](A_300_auth/README.md)

## 예시 문서

- [SCN.A.01 주문 생성 시나리오](.examples/SCN_A_01_place_order.md)

## 연관 문서

🏷️ 유스케이스 참조: [UC.A.01](../30-uc/.examples/UC_A_01_place_order.md) | API 참조: [API.A.01](../50-service-design/.examples/order/A_01_40-api/API_A_01_place_order.md) | 영속성 참조: [PST.A.01](../50-service-design/.examples/order/A_01_20-persistence/PST_A_01_order_persistence.md) | 서비스 참조: [SVC.A.01](../50-service-design/.examples/order/A_01_30-service/SVC_A_01_order_service.md)
