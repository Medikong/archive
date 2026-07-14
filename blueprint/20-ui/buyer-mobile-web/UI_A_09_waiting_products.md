---
id: UI.A.09
title: 기다리는 상품 랭킹 UI
type: ui-asset
status: draft
tags: [ui, ranking, wishlist, buyer, dropmong]
source: local
created: 2026-07-14
updated: 2026-07-14
---

# 기다리는 상품 랭킹 UI

## 기본 정보

- UI ID: `UI.A.09`
- 연관 Page: [PAGE.A.09](../../10-sitemap/buyer-mobile-web/PAGE_A_09_waiting_products.md)
- 진입점: 홈의 `기다리는 상품 > 전체 보기`
- 집계 기준: 드롭 상태와 무관한 누적 활성 찜 수 내림차순
- 최대 노출: 100위

## 에셋

![기다리는 상품 전체보기](assets/UI_A_09_waiting_products/UI_A_09_10_buyer_mobile_web.png)

## 화면 구성

| 영역 | 역할 | 상태/행동 |
| --- | --- | --- |
| 상단 앱 바 | 뒤로가기, 제목, 검색 진입 | 홈 복귀, 상품 검색 |
| 기준 안내 카드 | 랭킹의 의미와 갱신 기준 표시 | `누적 찜 기준 · 실시간 반영` |
| 랭킹 전환 탭 | 두 랭킹 전체보기 사이 이동 | 기다리는 상품 활성, 많이 보는 상품 이동 |
| Top 100 리스트 | 순위, 정돈된 상품 썸네일, 상품명, 가격, 찜 수 표시 | 카드 선택 시 상품 상세 이동 |

## 데이터와 예외

- `GET /v1/rankings/drops/upcoming`(`API.A.07-06`)을 `limit` 최대 100으로 호출한다.
- 동점은 현재 계약대로 `dropId` 오름차순으로 고정해 순위가 불필요하게 흔들리지 않게 한다.
- 목록이 비어 있으면 `아직 기다리는 상품이 없어요`와 홈 이동 CTA를 표시한다.
- 일부 카탈로그 표시 정보 조회가 실패하면 해당 행만 대체 썸네일과 최소 정보로 유지한다.

## 제작 근거

홈 시안의 보라색 강조색, 흰색 카드, 원형 순위 배지와 하트 지표를 그대로 확장했다. 랭킹 썸네일은 동일한 흰색 스튜디오 배경과 동일 비율의 제품 컷으로 통일했다.

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.07](../../00-requirements/REQ_A_07_interest_ranking.md) | 페이지 참조: [PAGE.A.09](../../10-sitemap/buyer-mobile-web/PAGE_A_09_waiting_products.md) | UC 참조: [UC.A.07](../../30-uc/UC_A_07_interest_ranking.md) | API 참조: [API.A.07-06](../../50-service-design/A_07_interest_ranking/A_07_40-api/README.md)

