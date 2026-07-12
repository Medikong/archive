---
id: WEB.A.02
title: 웹 상태와 데이터 전략
type: web-state-data-strategy
status: draft
tags: [web-application, state, server-state, url-state, tanstack-query, zustand, cache]
source: local
created: 2026-07-10
updated: 2026-07-10
---

# 웹 상태와 데이터 전략

## 기본 정보

- Web Application ID: `WEB.A.02`
- 적용 대상: [WEB.A.01](WEB_A_01_frontend_architecture.md)의 구매자, 판매자, 플랫폼 운영자 route
- 기본 원칙: 서버 권위 상태와 브라우저 표현 상태를 구분한다.
- 조회 도구 기준: Server Component 우선, 필요한 status polling에만 TanStack Query 적용
- 전역 store 기준: Zustand는 도입 조건을 충족할 때만 사용
- 화면 DTO와 웹 session 경계: [BFF.A.01](BFF_A_01_web_bff_module.md)

## 상태 분류

| 상태 유형 | 소유 위치 | 예시 | 기본 도구 |
| --- | --- | --- | --- |
| 서버 상태 | 업무 Context와 API | 드롭 상태, 오픈 시각, 재고, 가격, 주문, 결제, 쿠폰, 배송, 권한 | Server Component 조회, 제한된 TanStack Query |
| URL 상태 | 브라우저 URL | 검색어, 필터, 정렬, 페이지 번호, 선택 탭, 복귀 위치 | route param, `searchParams` |
| 폼 상태 | 해당 폼 | 로그인 입력, 배송지 선택, 쿠폰 코드, 드롭 등록 초안 | HTML form, React local state, 필요 시 form library |
| 로컬 UI 상태 | 해당 컴포넌트 | modal 열림, accordion, 현재 carousel index, 입력 도움말 표시 | `useState`, `useReducer` |
| 파생 상태 | 계산하는 컴포넌트 | 결제 버튼 활성 여부, 선택 수량 합계, 남은 표시 시간 | 렌더링 중 계산 |
| 공유 client 상태 | 제한적 provider | 여러 먼 Client Component가 함께 쓰는 임시 UI 작업 상태 | Zustand 도입 조건 통과 후 사용 |

한 값을 여러 분류에 동시에 저장하지 않는다. URL에서 계산할 수 있는 탭을 Zustand에도 넣거나, query cache의 주문 상태를 별도 store에 복제하지 않는다.

## 서버 권위 상태

다음 값은 브라우저 store, `localStorage`, cookie의 임의 값으로 판정하지 않는다.

- 드롭 오픈 전, 진행 중, 종료, 품절 상태
- 옵션별 판매 가능 수량과 재고 배정 성공 여부
- 상품 가격, 배송비, 쿠폰 적용 가능 여부, 최종 결제 금액
- 주문 생성, 결제 승인, 결제 실패, 취소, 환불 상태
- 배송 준비, 출고, 배송 중, 배송 완료 상태
- 로그인 session, role, permission, 위험 작업 승인 여부
- 판매자 검수, 운영자 승인, 재처리 결과

브라우저는 서버 응답을 표시하고 갱신할 뿐이다. 사용자에게 먼저 보여준 낙관적 상태가 주문, 결제, 재고의 최종 결과가 되어서는 안 된다.

## URL 상태

- 공유하거나 새로고침 뒤 복원해야 하는 검색어, 정렬, 필터, 페이지 번호는 URL에 둔다.
- 주문 ID, 드롭 ID, 판매자 ID처럼 리소스를 식별하는 값은 route param을 사용한다.
- 로그인 후 복귀 위치는 검증된 내부 경로만 허용하고 외부 URL은 거부한다.
- 비밀번호, 인증 코드, 결제 정보, 개인정보는 URL에 넣지 않는다.
- URL 변경은 브라우저 뒤로가기와 앞으로가기가 자연스럽게 동작하도록 `push`와 `replace`를 구분한다.

## 폼 상태

- 입력 중인 값과 validation message는 폼에 가장 가까운 컴포넌트가 소유한다.
- 서버 제출 시 입력을 다시 검증하고 session과 권한을 다시 확인한다.
- 서버가 계산하는 가격, 재고, 혜택을 hidden input 값으로 신뢰하지 않는다.
- 주문과 결제 제출은 버튼 비활성화만으로 중복을 막지 않고 멱등키를 함께 사용한다.
- 장시간 작성하는 판매자 드롭 등록 초안의 임시 저장은 브라우저 persistence가 아니라 서버 임시 저장 API를 우선한다.
- 실패 뒤 재입력 가능한 값과 폐기해야 하는 민감 값을 구분한다. 결제 credential과 인증 코드는 자동 복원하지 않는다.

## 로컬 UI와 파생 상태

- modal, tooltip, accordion처럼 한 화면 안에서 끝나는 값은 `useState` 또는 `useReducer`를 사용한다.
- props와 현재 state에서 계산할 수 있는 값은 별도 state와 `useEffect`로 동기화하지 않고 렌더링 중 계산한다.
- countdown은 `opensAt`과 서버가 제공한 `serverNow`의 차이를 기준으로 화면 숫자만 갱신한다. 구매 가능 여부는 API 응답으로 다시 판정한다.
- 컴포넌트가 사라질 때 함께 사라져야 하는 값은 전역 provider로 올리지 않는다.

## TanStack Query 적용 범위

TanStack Query는 모든 조회의 기본 도구가 아니다. 초기 화면 조회는 Server Component에서 처리하고, 화면을 보고 있는 동안 비동기 상태가 바뀌어 주기적 확인이 필요한 경우에만 사용한다.

| 대상 | Query key 예시 | polling 종료 조건 | 화면 처리 |
| --- | --- | --- | --- |
| 주문 처리 상태 | `['order-status', orderId]` | 성공, 실패, 취소, 만료 | 처리 단계, 실패 사유, 서버 기준 시각 표시 |
| 결제 처리 상태 | `['payment-status', paymentId]` | 승인, 거절, 취소, 만료 | 중복 결제 요청 없이 현재 결과만 갱신 |
| 배송 상태 | `['shipment-status', orderId]` | 배송 완료, 취소 | 마지막 배송 이벤트와 최신 시각 표시 |

다음 조회에는 우선 적용하지 않는다.

- 홈과 상품 상세의 최초 렌더링
- 인증 session과 권한 판정
- checkout의 가격과 재고 최종 검증
- 판매자/운영자 화면의 모든 목록을 일괄 client query로 전환하는 작업

## Polling 기준

- `refetchInterval`은 상태별 함수로 정의하고 terminal state에서 `false`를 반환해 중지한다.
- 브라우저 탭이 보이지 않을 때 polling을 계속하지 않는다. 다시 활성화되면 즉시 한 번 확인한다.
- 짧은 간격으로 시작해야 하는 처리 중 상태와 긴 간격으로 충분한 배송 상태를 같은 주기로 설정하지 않는다.
- polling 간격은 API rate limit, 예상 완료 시간, SLO를 근거로 정한다. 초기값은 운영 검증 전 설정값으로 분리한다.
- query 함수는 `AbortSignal`을 `fetch`에 전달해 route 이탈과 query 취소를 실제 HTTP 요청에 반영한다.
- WebSocket이나 SSE는 polling 부하와 사용자 지연이 실제 기준을 넘을 때 별도 결정으로 도입한다.

## 캐시 정책

| 캐시 위치 | 허용 대상 | 금지 또는 주의 대상 | 무효화 주체 |
| --- | --- | --- | --- |
| Next.js server cache | 공개 상품 설명, 종료된 드롭, 변경이 드문 공지 | 개인 주문, 결제, 재고 배정, 권한 | Server Action 또는 Route Handler의 tag/path 무효화 |
| TanStack Query cache | 보고 있는 주문, 결제, 배송의 짧은 상태 snapshot | 장기 보존, 다른 사용자의 데이터, 최종 권위 상태 | mutation 성공 후 정확한 query key 무효화, polling |
| 브라우저 storage | 민감하지 않은 화면 환경설정 | token, 개인정보, 주문/결제/재고/권한 | 버전이 있는 명시적 migration과 만료 정책 |

- 개인화되고 정합성이 중요한 응답은 캐시하지 않는 것을 기본값으로 둔다.
- 같은 응답을 Next.js server cache와 TanStack Query에 동시에 오래 보관하지 않는다. 각 캐시의 목적과 최신성 기준을 문서에 적는다.
- 상품이나 공지처럼 약간 늦게 갱신되어도 되는 공개 데이터는 tag 기반 재검증을 사용한다.
- 사용자가 방금 변경한 결과를 즉시 봐야 하면 read-your-own-writes가 필요한 tag 갱신 또는 정확한 client query 무효화를 사용한다.
- `router.refresh()`는 현재 route의 Client Router Cache를 새로 읽게 하지만 server cache 무효화를 대신하지 않는다.
- Kubernetes에서 Next.js 인스턴스를 둘 이상 실행할 때 tag 무효화를 사용하려면 인스턴스 사이 tag 상태와 cache를 공유해야 한다. 공유 방식을 마련하지 않으면 correctness가 중요한 데이터에는 server cache를 사용하지 않는다.

## 최신 시각과 지연 표시

| 값 | 의미 | 표시 용도 |
| --- | --- | --- |
| `sourceUpdatedAt` | 업무 서비스가 상태를 확정한 시각 | 주문, 결제, 배송의 기준 시각 |
| `serverNow` | 응답을 만든 서버 기준 현재 시각 | countdown과 clock skew 보정 |
| `fetchedAt` | 웹이 응답을 받은 시각 | 마지막 조회 시각과 네트워크 지연 확인 |
| `stale` | 서비스가 최신성을 보장하지 못함 | 지연 배지, 재조회, 제한된 화면 안내 |

- 화면의 `마지막 갱신`은 가능하면 `sourceUpdatedAt`을 사용한다. 브라우저가 받은 시각을 업무 상태 변경 시각처럼 표시하지 않는다.
- polling 응답 순서가 뒤바뀌면 더 오래된 `sourceUpdatedAt` 결과로 화면을 되돌리지 않는다.
- 서비스가 cached snapshot을 반환하면 원천 시각과 stale 여부를 함께 제공해야 한다.
- 드롭 오픈 시각은 사용자 기기 시간이 아니라 서버 판정을 따른다.

## 재시도 정책

| 요청 | 자동 재시도 | 기준 |
| --- | --- | --- |
| 공개/상태 GET | 제한적으로 허용 | 네트워크 오류와 일시적 `5xx`만 지수 backoff와 jitter로 재시도 |
| 인증·권한 실패 | 금지 | `401`, `403`을 재시도하지 않고 로그인 또는 권한 안내 |
| 입력·업무 조건 실패 | 금지 | `4xx`, 품절, 만료, 조건 미충족 사유를 표시 |
| 주문·결제 mutation | 기본 금지 | 멱등키가 있고 API가 동일 결과 반환을 보장한 경우에만 명시적 재시도 |
| 조회 polling | 다음 주기에서 재확인 | 연속 실패 횟수에 따라 간격을 늘리고 화면에 지연 표시 |

- query의 자동 재시도 횟수와 지연은 전역 기본값에 맡기지 않고 상태 query별로 적는다.
- 결제 버튼을 다시 누르게 하기 전에 이전 요청 결과를 조회한다.
- mutation 실패를 성공처럼 처리하거나 이전 cache 값을 최신 결과로 가장하지 않는다.
- 사용자가 재시도할 때 trace ID와 멱등키의 관계를 유지한다.

## Mutation과 무효화

- mutation이 성공하면 화면 전체 cache를 비우지 않고 영향을 받는 query key만 무효화한다.
- 드롭 정보 변경은 해당 드롭 상세와 목록 tag를, 주문 생성은 해당 주문 상태와 구매자 주문 목록을 갱신한다.
- 주문과 결제는 낙관적 성공 상태를 쓰지 않는다. `접수됨`과 `최종 성공`을 별도 상태로 보여준다.
- 여러 query를 갱신해야 하면 서로 독립적인 무효화 작업을 함께 시작하고 완료 시점을 명확히 기다린다.
- Server Action은 API endpoint와 같은 공개 mutation 진입점으로 취급해 입력, 인증, 권한, 멱등성을 action 내부에서 검증한다.

## Zustand 도입 조건

초기에는 Zustand를 설치하지 않는다. 다음 질문을 모두 통과하는 상태가 생길 때 도입한다.

1. 서버 권위 데이터나 query cache 데이터가 아닌가?
2. URL로 공유하거나 복원할 상태가 아닌가?
3. 하나의 폼이나 가까운 컴포넌트의 local state로 끝나지 않는가?
4. route group 안의 여러 먼 Client Component가 같은 임시 UI 상태를 자주 읽고 쓰는가?
5. Context와 reducer만으로 관리할 때 실제 re-render 또는 유지보수 문제가 측정되었는가?

도입 시 다음 규칙을 지킨다.

- module 전역 store를 서버 요청 사이에 공유하지 않는다. 필요한 route 또는 액터 provider에서 store instance를 만든다.
- React Server Component는 store를 읽거나 쓰지 않는다.
- selector는 필요한 최소 값만 구독한다.
- 로그인 종료, 사용자 전환, route 이탈 같은 reset 조건을 정의한다.
- persistence는 기본으로 끈다. 꼭 필요하면 key version, 만료, migration을 두고 최소한의 비민감 값만 저장한다.
- token, session 원문, 개인정보, 주문, 결제, 재고, 권한을 저장하지 않는다.

도입 기록에는 상태 소유자, 생명주기, reset 조건, persistence 여부, URL·폼·query cache로 해결할 수 없었던 이유를 남긴다.

## 페이지별 적용 예

| 페이지 | 초기 데이터 | Client 상태 | 갱신 방식 |
| --- | --- | --- | --- |
| [PAGE.A.01](../10-sitemap/buyer-mobile-web/PAGE_A_01_homepage.md) 홈 | Server Component에서 공개 드롭 조회 | carousel index, 알림 신청 버튼 상태 | 사용자 mutation 후 관련 드롭만 갱신 |
| [PAGE.A.02](../10-sitemap/buyer-mobile-web/PAGE_A_02_product_detail.md) 상품 상세 | 상품, 드롭, 옵션 snapshot | 옵션 선택, 수량, countdown 표시 | 구매 직전 서버 재검증 |
| [PAGE.A.11](../10-sitemap/buyer-mobile-web/PAGE_A_11_payment.md) 주문/결제 | checkout snapshot, 배송지, 결제수단 표시 정보 | 입력, 동의, 제출 중 상태 | mutation 후 주문/결제 status polling 시작 |
| [PAGE.A.14](../10-sitemap/buyer-mobile-web/PAGE_A_14_order_complete.md) 주문 완료 | 주문 ID와 최초 상태 | 주문/결제 status query | terminal state에서 polling 중지 |
| [PAGE.A.16](../10-sitemap/buyer-mobile-web/PAGE_A_16_track_order.md) 배송 조회 | 배송 snapshot | 배송 status query | 배송 완료에서 polling 중지 |
| [PAGE.A.200](../10-sitemap/PAGE_A_200_seller_portal/README.md) 판매자 대시보드 | 서버에서 권한과 요약 조회 | 필터와 펼침 상태 | 진행 중 드롭만 제한적 갱신 |

## 오류와 사용자 피드백

- 최초 조회 실패는 가까운 route `error.tsx`가 처리하고 trace ID와 재시도를 제공한다.
- polling 실패 중 이전 성공 데이터가 있으면 데이터를 숨기지 않고 기준 시각과 지연 경고를 함께 표시한다.
- 빈 목록은 성공 응답이므로 `EmptyState`와 다음 행동을 표시한다.
- 품절, 결제 거절, 인증 만료는 기술 오류가 아니라 업무 상태로 표시한다.
- 연속 실패로 최신성을 보장할 수 없으면 위험한 mutation CTA를 비활성화하거나 서버 재검증을 먼저 수행한다.

## 검증 기준

- 주문, 결제, 재고, 배송, 권한 상태가 Zustand나 browser storage에 권위 상태로 저장되지 않는다.
- 검색, 정렬, 필터, 페이지 번호가 URL로 복원된다.
- polling은 terminal state와 route 이탈에서 중지되고 숨겨진 탭에서 불필요하게 계속되지 않는다.
- `4xx`와 업무 실패는 자동 재시도하지 않는다.
- 주문과 결제 mutation은 멱등키 없이 자동 재시도하지 않는다.
- 화면에 업무 상태 변경 시각과 웹 조회 시각을 구분해 표시한다.
- 여러 Next.js 인스턴스에서 server cache를 쓸 경우 무효화 전달 방식을 검증한다.

## 연관 태그

🏷️ 요구사항 참조: [REQ.A.01](../00-requirements/REQ_A_01_limited_drop_commerce.md), [REQ.A.03](../00-requirements/REQ_A_03_seller.md), [REQ.A.04](../00-requirements/REQ_A_04_platform_operator_admin.md), [REQ.A.05](../00-requirements/REQ_A_05_auth_member.md) | 페이지 참조: [사이트맵 인덱스](../10-sitemap/README.md) | UI 참조: [UI 인덱스](../20-ui/README.md) | UC 참조: [UC.A.01](../30-uc/UC_A_01_buyer_purchase_delivery.md), [UC.A.02](../30-uc/UC_A_02_seller_manage_drop.md), [UC.A.03](../30-uc/UC_A_03_platform_operator_admin.md) | BFF 참조: [BFF.A.01](BFF_A_01_web_bff_module.md) | 서비스 참조: [서비스 상세 설계](../50-service-design/README.md) | 시나리오 참조: [처리 시퀀스](../80-sequence/README.md) | 웹 구조 참조: [WEB.A.01](WEB_A_01_frontend_architecture.md)

## 확인 필요

- 주문, 결제, 배송 상태 API의 terminal state와 `sourceUpdatedAt` 필드를 확정한다.
- API rate limit과 SLO를 기준으로 상태별 polling 간격과 최대 대기 시간을 정한다.
- Next.js server cache를 공유할 저장소와 tag 무효화 전달 방식을 정한다.
- 판매자 드롭 등록 초안의 서버 임시 저장 주기와 만료 정책을 정한다.
- 플랫폼 운영자 화면이 추가되면 실시간 업무 지표의 polling, SSE, WebSocket 선택 기준을 별도 검토한다.

## 참고 자료

- [Next.js 데이터 재검증](https://nextjs.org/docs/app/getting-started/revalidating)
- [Next.js self-hosting과 다중 인스턴스 캐시](https://nextjs.org/docs/app/guides/self-hosting)
- [TanStack Query `useQuery`의 `refetchInterval`](https://tanstack.com/query/latest/docs/framework/react/reference/useQuery)
- [TanStack Query query invalidation](https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation)
- [Zustand의 Next.js 사용 기준](https://zustand.docs.pmnd.rs/learn/guides/nextjs)
