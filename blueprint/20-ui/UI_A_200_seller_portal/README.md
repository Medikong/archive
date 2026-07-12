---
id: UI.A.200
group_id: UI.A.200
ui_ids: [UI.A.200, UI.A.201, UI.A.202, UI.A.203, UI.A.204, UI.A.205, UI.A.206, UI.A.207, UI.A.208, UI.A.209, UI.A.210, UI.A.211]
page_ids: [PAGE.A.200, PAGE.A.201, PAGE.A.202, PAGE.A.203, PAGE.A.204, PAGE.A.205, PAGE.A.206, PAGE.A.207, PAGE.A.208, PAGE.A.209, PAGE.A.210, PAGE.A.211]
title: 판매자 웹 포털 UI
type: ui-asset
status: draft
device: desktop-web
viewports: [1440, 1024, 768]
tags: [ui, seller, seller-portal, desktop-web, dashboard, screen-set, components, dropmong]
source: generated
created: 2026-07-10
updated: 2026-07-10
---

# 판매자 웹 포털 UI

## 문서 역할

이 문서는 `PAGE.A.200~211` 판매자 웹 포털의 시각 방향, 페이지별 화면 시안, 공통 컴포넌트와 상호작용 기준을 연결한다.

`UI.A.200~211`은 같은 데스크톱 웹 셸을 공유한다. 각 이미지는 페이지별 정보 우선순위와 대표 상태를 보여주고, 컴포넌트 시트는 화면 사이에서 재사용할 UI 규칙을 정리한다.

## 기본 정보

- UI ID: `UI.A.200~211`
- 기준 페이지: [PAGE.A.200~211 판매자 웹 포털](../../10-sitemap/PAGE_A_200_seller_portal/README.md)
- 기준 요구사항: [REQ.A.03 판매자](../../00-requirements/REQ_A_03_seller.md)
- 기준 유스케이스: [UC.A.02 판매자 드롭 운영](../../30-uc/UC_A_02_seller_manage_drop.md)
- 대상 환경: 데스크톱 우선 반응형 웹
- 생성 참고 자료: `dropmong-ui-ux-component-sheet.png`, `dropmong-brand.png`

## UI 시안

### 페이지 화면 모음

<div style="display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 14px; width: 100%; align-items: start; padding: 8px 0 20px;">
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_200_01_seller_portal_dashboard.png" alt="UI.A.200 판매자 대시보드" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.200<br/>판매자 대시보드</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_201_01_drop_management.png" alt="UI.A.201 드롭 관리" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.201<br/>드롭 관리</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_202_01_product_management.png" alt="UI.A.202 상품 관리" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.202<br/>상품 관리</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_203_01_drop_create_edit.png" alt="UI.A.203 드롭 등록 및 편집" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.203<br/>드롭 등록·편집</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_204_01_review_change_request.png" alt="UI.A.204 검수 및 변경 요청" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.204<br/>검수·변경 요청</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_205_01_order_fulfillment.png" alt="UI.A.205 주문 및 출고" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.205<br/>주문·출고</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_206_01_coupon_promotion.png" alt="UI.A.206 쿠폰 및 프로모션" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.206<br/>쿠폰·프로모션</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_207_01_sales_analytics.png" alt="UI.A.207 판매 분석" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.207<br/>판매 분석</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_208_01_settlement.png" alt="UI.A.208 정산 조회" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.208<br/>정산 조회</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_209_01_store_settings.png" alt="UI.A.209 판매자 및 스토어 정보" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.209<br/>판매자·스토어 정보</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_210_01_team_permissions.png" alt="UI.A.210 팀 및 권한" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.210<br/>팀·권한</figcaption></figure>
  <figure style="margin: 0; min-width: 0;"><img src="../assets/UI_A_200_seller_portal/UI_A_211_01_operational_issues.png" alt="UI.A.211 운영 이슈" style="display: block; width: 100%; height: auto;" /><figcaption>UI.A.211<br/>운영 이슈</figcaption></figure>
</div>

### 판매자 웹 컴포넌트 시트

![DropMong 판매자 웹 컴포넌트 시트](../assets/UI_A_200_seller_portal/UI_A_200_02_seller_portal_component_sheet.png)

## 에셋 목록

| 에셋 | 역할 | 적용 범위 |
| --- | --- | --- |
| [판매자 대시보드](../assets/UI_A_200_seller_portal/UI_A_200_01_seller_portal_dashboard.png) | 데스크톱 웹 셸과 대시보드 정보 우선순위 시안 | `PAGE.A.200` |
| [판매자 웹 컴포넌트 시트](../assets/UI_A_200_seller_portal/UI_A_200_02_seller_portal_component_sheet.png) | 내비게이션, 입력, 표, 카드, 상태, 차트, 권한 UI 후보 | `PAGE.A.200~211` |
| [드롭 관리](../assets/UI_A_200_seller_portal/UI_A_201_01_drop_management.png) | 상태별 드롭 탐색, 재고·주문 현황과 후속 작업 | `PAGE.A.201` |
| [상품 관리](../assets/UI_A_200_seller_portal/UI_A_202_01_product_management.png) | 상품 초안, 옵션·가격·재고와 구매자 노출 미리보기 | `PAGE.A.202` |
| [드롭 등록·편집](../assets/UI_A_200_seller_portal/UI_A_203_01_drop_create_edit.png) | 상품 정보, 판매 조건, 재고와 검수 요청 단계 | `PAGE.A.203` |
| [검수·변경 요청](../assets/UI_A_200_seller_portal/UI_A_204_01_review_change_request.png) | 반려 사유, 검수 이력과 승인 후 변경 전후 비교 | `PAGE.A.204` |
| [주문·출고](../assets/UI_A_200_seller_portal/UI_A_205_01_order_fulfillment.png) | 주문 표, 출고 상태와 감사 대상 자료 생성 | `PAGE.A.205` |
| [쿠폰·프로모션](../assets/UI_A_200_seller_portal/UI_A_206_01_coupon_promotion.png) | 판매자 쿠폰과 제휴 쿠폰 제안·성과 | `PAGE.A.206` |
| [판매 분석](../assets/UI_A_200_seller_portal/UI_A_207_01_sales_analytics.png) | 구매 전환, 상품·옵션·쿠폰 효과 분석 | `PAGE.A.207` |
| [정산 조회](../assets/UI_A_200_seller_portal/UI_A_208_01_settlement.png) | 예정·보류·확정·차감 금액과 사유 | `PAGE.A.208` |
| [판매자·스토어 정보](../assets/UI_A_200_seller_portal/UI_A_209_01_store_settings.png) | 판매자 정보, 브랜딩, 배송·교환·환불 정책 | `PAGE.A.209` |
| [팀·권한](../assets/UI_A_200_seller_portal/UI_A_210_01_team_permissions.png) | 구성원, 역할별 권한과 변경 이력 | `PAGE.A.210` |
| [운영 이슈](../assets/UI_A_200_seller_portal/UI_A_211_01_operational_issues.png) | 판매자 귀책 이슈 신고, 조회와 처리 상태 | `PAGE.A.211` |

## 시각 방향

- 구매자 앱의 보라색, 라벤더, 핑크, 코랄 포인트를 유지하되 업무용 웹에서는 흰색 표면과 잉크 색상 비중을 높인다.
- 주요 작업은 `#6C3DF5`를 쓰고, 위험·반려·오류는 코랄/레드, 대기·주의는 앰버, 성공은 그린 계열로 구분한다.
- 판매 성과보다 먼저 처리해야 하는 검수 반려, 출고 지연, 권한 오류를 대시보드 상단에서 드러낸다.
- 장식용 그라데이션은 브랜드 영역과 핵심 CTA로 제한하고, 표와 입력 영역은 대비와 판독성을 우선한다.
- 마스코트는 빈 상태나 도움말처럼 친근함이 필요한 곳에만 작게 사용한다.

## 웹 셸

| 영역 | 구성 | 동작 |
| --- | --- | --- |
| 좌측 사이드바 | 로고, 판매자 표시, 대시보드, 드롭, 상품, 주문·출고, 쿠폰, 판매 분석, 정산, 스토어, 팀·권한 | 현재 메뉴를 강조하고 역할에 따라 메뉴를 숨기거나 읽기 전용 처리한다. |
| 상단 바 | 검색, 알림, 도움말, 판매자/스토어 선택, 사용자 메뉴 | 판매자 계정 전환과 현재 권한을 확인한다. |
| 페이지 머리말 | 페이지 제목, 설명, 최신성, 기간·상태 필터, 주요 CTA | 한 화면의 대표 작업을 하나만 강하게 표시한다. |
| 본문 그리드 | KPI, 처리할 작업, 표, 카드, 차트 | 화면 폭에 따라 12열에서 8열, 4열로 재배치한다. |
| 상세 패널·모달 | 검수 이력, 변경 비교, 다운로드 목적, 확인 대화상자 | 감사가 필요한 작업의 이유와 결과를 남긴다. |

## 대시보드 화면 구성

| 번호 | 영역 | 화면 요소 | 사용자 행동 |
| --- | --- | --- | --- |
| 1 | 페이지 머리말 | 환영 문구, 집계 기준 시각, `새 드롭 만들기` CTA | 드롭 등록 시작 |
| 2 | 확인 필요한 작업 | 검수 반려, 출고 지연, 제휴 쿠폰 응답, 정산 보류 알림 | 관련 페이지로 이동 |
| 3 | KPI 카드 | 진행 중 드롭, 오늘 주문, 결제 성공률, 출고 대기 | 정의·집계 시각 확인, 상세 분석 이동 |
| 4 | 진행 중인 드롭 | 상품, 상태, 카운트다운, 재고 진행률, 주문 성공 수 | 드롭 상세 또는 상태 조회 |
| 5 | 판매 현황 | 조회, 알림 신청, 구매 시도, 주문 성공 차트 | 기간·지표 변경, 분석 이동 |
| 6 | 최근 주문 | 주문 번호, 상품, 마스킹 구매자, 출고 상태, 금액 | 주문 상세, 출고 처리 |
| 7 | 예정 일정 | 검수 제출, 오픈 예정, 출고 마감 일정 | 관련 작업 확인 |

## 페이지와 UI 연결

| Page ID | UI에서 확인할 주요 요소 | 현재 근거 |
| --- | --- | --- |
| `PAGE.A.200` | 전체 웹 셸, KPI, 작업 알림, 진행 드롭, 차트, 최근 주문 | 대시보드 시안 |
| `PAGE.A.201` | 상태 필터, 드롭 목록 카드/표, 카운트다운, 재고 진행률 | [UI.A.201](../assets/UI_A_200_seller_portal/UI_A_201_01_drop_management.png) |
| `PAGE.A.202` | 상품 검색, 이미지 업로드, 옵션 입력, 구매자 노출 미리보기 | [UI.A.202](../assets/UI_A_200_seller_portal/UI_A_202_01_product_management.png) |
| `PAGE.A.203` | 상품 정보·판매 조건·재고·검수 요청 단계 표시 | [UI.A.203](../assets/UI_A_200_seller_portal/UI_A_203_01_drop_create_edit.png) |
| `PAGE.A.204` | 검수 상태 배지, 반려 알림, 변경 요청 비교 | [UI.A.204](../assets/UI_A_200_seller_portal/UI_A_204_01_review_change_request.png) |
| `PAGE.A.205` | 주문 표, 출고 상태, 개인정보 마스킹, 다운로드 목적 | [UI.A.205](../assets/UI_A_200_seller_portal/UI_A_205_01_order_fulfillment.png) |
| `PAGE.A.206` | 쿠폰 상태, 할인 조건, 제휴 제안, 성과 카드 | [UI.A.206](../assets/UI_A_200_seller_portal/UI_A_206_01_coupon_promotion.png) |
| `PAGE.A.207` | KPI, 기간 필터, 구매 전환·시계열 차트, 집계 기준 시각 | [UI.A.207](../assets/UI_A_200_seller_portal/UI_A_207_01_sales_analytics.png) |
| `PAGE.A.208` | 정산 상태 카드, 금액 표, 보류·차감 사유 | [UI.A.208](../assets/UI_A_200_seller_portal/UI_A_208_01_settlement.png) |
| `PAGE.A.209` | 판매자·스토어 폼, 이미지 업로드, 구매자 노출 미리보기 | [UI.A.209](../assets/UI_A_200_seller_portal/UI_A_209_01_store_settings.png) |
| `PAGE.A.210` | 구성원 목록, 역할 선택, 메뉴 권한 표, 변경 이력 | [UI.A.210](../assets/UI_A_200_seller_portal/UI_A_210_01_team_permissions.png) |
| `PAGE.A.211` | 이슈 유형, 사유 입력, 처리 상태와 타임라인 | [UI.A.211](../assets/UI_A_200_seller_portal/UI_A_211_01_operational_issues.png) |

## 공통 컴포넌트

| 컴포넌트 | 변형 | 사용 기준 |
| --- | --- | --- |
| 사이드바 항목 | 기본, 활성, 읽기 전용, 잠금, 축소 | 메뉴 권한과 현재 위치를 함께 표현한다. |
| 버튼 | 주요, 보조, 텍스트, 위험, 아이콘 | 화면당 주요 버튼은 하나를 원칙으로 한다. |
| 입력 | 텍스트, 숫자, 금액, 수량, 날짜·시각, 선택, 파일 업로드 | 오류 원인과 수정 방법을 입력 가까이에 표시한다. |
| 필터 | 검색, 상태, 기간, 상품, 드롭, 쿠폰, 초기화 | 선택한 필터와 결과 건수를 함께 표시한다. |
| 상태 배지 | 임시 저장, 검수 중, 승인, 반려, 진행 중, 종료 | 색상만으로 상태를 구분하지 않고 문구를 병기한다. |
| KPI 카드 | 값, 전기 대비, 최신성, 상세 링크 | 실시간 추정과 집계 완료를 구분한다. |
| 드롭 카드 | 상품, 오픈 시각, 상태, 재고, 주문, 작업 | 카드 전체 클릭과 개별 작업 버튼을 충돌시키지 않는다. |
| 데이터 표 | 선택, 정렬, 고정 열, 페이지네이션, 빈 상태 | 대량 주문·상품·드롭 조회에 사용한다. |
| 차트 | 선, 막대, 범례, 툴팁, 비교 기간 | 원값, 단위, 집계 기준 시각을 확인할 수 있어야 한다. |
| 단계 표시 | 상품 정보, 판매 조건, 재고, 검수 요청 | 현재 단계, 완료 단계, 오류 단계를 구분한다. |
| 권한 표 | 역할 행, 메뉴·작업 열, 읽기·쓰기 체크 | 대표 관리자만 저장할 수 있다. |
| 알림 | 성공, 정보, 주의, 오류 | 다음 행동이 있으면 링크나 버튼을 함께 제공한다. |
| 확인 모달 | 일반 확인, 변경 요청, 위험 작업, 다운로드 목적 | 대상, 영향, 사유, 결과를 명확히 표시한다. |

## 화면에 필요한 정보

공통 컨텍스트와 각 페이지가 조회·입력하는 필드를 분리한다. 목록형 필드는 한 행의 표시값과 필터·페이지네이션에 필요한 값을 함께 정의하며, 집계값은 기준 시각과 확정 여부를 반드시 포함한다.

### 공통 컨텍스트

| 화면 영역 | 필드 | 타입 | 용도 |
| --- | --- | --- | --- |
| 판매자 식별 | `seller.id` | string | 모든 조회·수정 요청의 판매자 범위를 고정 |
| 판매자 식별 | `seller.displayName` | string | 상단 바와 계정 전환 영역에 현재 판매자명 표시 |
| 판매자 식별 | `seller.type` | enum | 셀러·브랜드·제휴 판매자 등 계약 유형 표시 |
| 판매자 상태 | `seller.status` | enum | 정상, 검증 대기, 이용 제한, 탈퇴 상태 표시 |
| 판매자 상태 | `seller.verificationStatus` | enum | 사업자 및 판매자 인증 완료 여부 표시 |
| 사용자 식별 | `member.id` | string | 현재 로그인한 판매자 구성원 식별 |
| 사용자 식별 | `member.displayName` | string | 상단 프로필과 감사 기록에 사용자명 표시 |
| 사용자 권한 | `member.role` | enum | 대표 관리자, 상품 담당자, 출고 담당자, 성과 조회자 구분 |
| 사용자 권한 | `member.permissions[]` | enum[] | 페이지와 입력 항목의 조회·수정 가능 여부 결정 |
| 전역 알림 | `navigation.unreadNotificationCount` | number | 상단 알림 아이콘에 읽지 않은 알림 수 표시 |
| 전역 알림 | `navigation.notifications[].targetPageId` | page-id | 알림 선택 시 이동할 실제 판매자 페이지 지정 |

## 화면에서 지원하는 행동

- 사용자는 역할에 허용된 메뉴만 확인하고 판매자 계정 범위 안에서 작업한다.
- 사용자는 대시보드에서 검수 반려, 출고 지연, 제휴 쿠폰 응답 같은 우선 작업으로 바로 이동한다.
- 사용자는 상품과 드롭을 등록하고 임시 저장한 뒤 검수를 요청한다.
- 사용자는 승인 후 핵심 조건을 직접 수정하지 않고 변경 요청을 제출한다.
- 사용자는 주문 자료를 내려받기 전에 목적과 범위를 확인한다.
- 사용자는 실시간 추정 지표와 집계 완료 지표의 기준 시각을 확인한다.
- 사용자는 팀 구성원의 역할별 메뉴·작업 권한을 확인하고 대표 관리자 권한으로 변경한다.

## 상태와 오류 처리

| 상태 | 화면 처리 |
| --- | --- |
| 권한 없음 | 메뉴 숨김 또는 읽기 전용 처리 후 접근 거부 사유와 관리자 문의 경로를 표시한다. |
| 다른 판매자 데이터 요청 | 일반 오류로 감추지 않고 권한 오류를 기록하며 현재 판매자 컨텍스트를 다시 확인하게 한다. |
| 검수 반려 | 반려 배지, 사유, 수정 필요 항목과 재제출 CTA를 표시한다. |
| 승인 후 변경 | 직접 저장을 막고 변경 전후 값, 사유, 검토 상태를 가진 요청으로 전환한다. |
| 오픈 임박·진행 중 | 삭제, 가격 변경, 수량 축소, 비공개 전환을 제한하고 이유를 표시한다. |
| 통계 집계 지연 | 마지막 갱신 시각, 실시간 추정 여부와 재시도 상태를 표시한다. |
| 부분 장애 | 사용 가능한 조회 범위와 제한된 작업을 명시하고 성공처럼 보이는 대체 값을 사용하지 않는다. |
| 다운로드 실패 | 파일 생성 상태, 만료, 재시도 가능 시각과 감사 기록 여부를 표시한다. |

## 반응형 기준

| 너비 | 레이아웃 |
| --- | --- |
| 1440px 이상 | 확장 사이드바와 12열 본문, KPI 4열, 차트와 표 병렬 배치 |
| 1024~1439px | 축소 사이드바와 8열 본문, KPI 2열, 차트와 표 세로 배치 |
| 768~1023px | 아이콘 사이드바 또는 드로어, 4열 본문, 표 가로 스크롤, 보조 패널 전체 너비 |
| 767px 이하 | 핵심 조회와 승인 상태 확인만 보조 지원하며 대량 등록·권한 편집은 데스크톱 사용을 안내한다. |

## 접근성 기준

- 본문 텍스트와 배경은 WCAG AA 수준의 명도 대비를 확보한다.
- 상태는 색상뿐 아니라 배지 문구, 아이콘, 보조 설명으로 구분한다.
- 사이드바, 표, 필터, 모달은 키보드로 이동하고 실행할 수 있어야 한다.
- 표 머리글과 정렬 상태, 차트 요약, 오류 문구를 보조 기술에 제공한다.
- 위험 작업 확인 시 초점을 모달 안에 가두고 닫은 뒤 원래 실행 요소로 돌려보낸다.

## 생성 기준

- 기본 이미지 생성 도구로 새 시안을 만들었다.
- `dropmong-ui-ux-component-sheet.png`는 색상, 타이포그래피, 버튼, 입력, 배지, 카드와 아이콘의 시각 기준으로 사용했다.
- `UI_A_200_01_seller_portal_dashboard.png`는 모든 화면의 사이드바, 상단 바, 그리드와 정보 밀도 기준으로 사용했다.
- `dropmong-brand.png`는 로고, 보라·라벤더 분위기, 코랄 포인트와 친근한 브랜드 인상의 기준으로 사용했다.
- 생성 이미지의 세부 문구보다 이 문서의 페이지·필드·상태 정의를 구현 기준으로 삼는다.

## 화면 생성 프롬프트 요약

| UI ID | 생성 요청의 핵심 |
| --- | --- |
| `UI.A.201` | 상태별 드롭 목록, 재고 진행률, 오픈 시각과 후속 작업을 가진 드롭 관리 화면 |
| `UI.A.202` | 상품 초안, 옵션·가격·재고와 구매자 노출 미리보기를 가진 상품 관리 화면 |
| `UI.A.203` | 상품 정보, 판매 조건, 재고, 검수 요청의 네 단계와 실시간 요약을 가진 등록 화면 |
| `UI.A.204` | 반려 사유, 검수 이력, 변경 전후 비교와 재요청 작업을 가진 검수 화면 |
| `UI.A.205` | 개인정보를 마스킹한 주문 표와 출고 자료 생성 목적을 확인하는 주문·출고 화면 |
| `UI.A.206` | 판매자 쿠폰과 제휴 쿠폰 제안, 비용 부담, 사용 성과를 함께 보는 쿠폰 화면 |
| `UI.A.207` | 구매 전환, 상품·옵션·쿠폰 효과와 집계 최신성을 보여주는 판매 분석 화면 |
| `UI.A.208` | 정산 예정·보류·확정·차감 금액과 보류 사유를 구분하는 정산 화면 |
| `UI.A.209` | 판매자·스토어 정보와 구매자에게 보이는 미리보기를 함께 편집하는 설정 화면 |
| `UI.A.210` | 구성원 목록, 역할별 권한 표와 변경 이력을 가진 팀 관리 화면 |
| `UI.A.211` | 취소·지연·재고·상품 정보 오류의 신고와 처리 상태를 확인하는 운영 이슈 화면 |

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.03](../../00-requirements/REQ_A_03_seller.md), [REQ.A.02](../../00-requirements/REQ_A_02_coupon_benefit.md) | 페이지 참조: [PAGE.A.200~211](../../10-sitemap/PAGE_A_200_seller_portal/README.md) | UI 참조: UI.A.200~211 | UC 참조: [UC.A.02](../../30-uc/UC_A_02_seller_manage_drop.md) | 영속성 참조: PST.A.200 예정 | 서비스 참조: SVC.A.200 예정 | 시나리오 참조: SCN.A.200 예정 | API 참조: API.A.200 예정

## 확인 필요

- 판매자 전용 로그인 시안과 판매자 역할 선택 방식을 확정한다.
- 상품과 드롭 등록을 한 단계형 화면으로 합칠지 별도 화면으로 유지할지 사용성 검증을 진행한다.
- 역할별 사이드바 노출, 읽기 전용, 작업 버튼 제한 기준을 확정한다.
- 대량 주문 표의 고정 열, 일괄 출고, 다운로드 파일 생성 방식을 정한다.
- 실시간 지표의 허용 지연과 집계 완료 표시 문구를 정한다.
- 페이지별 시안을 하나의 UI 그룹 문서에 유지할지, 구현 단계에서 개별 `UI` 문서로 분리할지 결정한다.
