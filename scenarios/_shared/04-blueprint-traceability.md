# 구매 시나리오 Blueprint 추적성

작성일: 2026-07-14

이 문서는 blueprint의 목표 설계와 구매 시나리오의 구현·검증 문서를 연결한다. 요구사항을 복사하지 않고 원문 ID와 현재 검증 경계만 관리한다.

## 1. 문서 책임

| 계층 | 원장 |
| --- | --- |
| 제품 요구사항·화면·유스케이스 | `archive/blueprint` |
| 현재 구현 흐름과 수용 테스트 | `archive/scenarios` |
| API·이벤트 기계 계약 | `services/contracts` |
| 실제 동작과 실행 테스트 | `services` |

## 2. 시나리오 연결표

| 시나리오 | 요구사항 | 화면 | 유스케이스 | E2E | 현재 판정 |
| --- | --- | --- | --- | --- | --- |
| 정상 구매 | `REQ.A.01` | `PAGE.A.02`, `PAGE.A.11` | `UC.A.01` 정상 흐름 | `04-customer-drop-purchase-happy-path` | 내부 백엔드 완료 |
| 결제 실패 | `REQ.A.01.FR-009`, `FR-011`, `FR-012`, `FR-013` | `PAGE.A.11` | `UC.A.01` 결제 예외 | `05-payment-failure-flow` | 부분 완료: 즉시 실패·재고 회복 완료, 지연·만료·실패 알림 미완료 |
| 품절·동시성 | `REQ.A.01.FR-006`, `FR-007`, `FR-008`, `NFR-002` | `PAGE.A.02` | `UC.A.01` 품절 예외 | `06-sold-out-concurrency-flow` | oversell 방지 완료, admission·spike 미완료 |

## 3. 기준 문서

| ID | 경로 |
| --- | --- |
| `REQ.A.01` | `../../blueprint/00-requirements/REQ_A_01_limited_drop_commerce.md` |
| `PAGE.A.02` | `../../blueprint/10-sitemap/buyer-mobile-web/PAGE_A_02_product_detail.md` |
| `PAGE.A.11` | `../../blueprint/10-sitemap/buyer-mobile-web/PAGE_A_11_payment.md` |
| `UC.A.01` | `../../blueprint/30-uc/UC_A_01_buyer_purchase_delivery.md` |

## 4. 변경 영향 규칙

- `REQ.A.01`이 바뀌면 세 시나리오의 상세 설계와 테스트 시나리오를 확인한다.
- `PAGE.A.02`가 바뀌면 정상 구매와 품절·동시성 사용자 여정을 확인한다.
- `PAGE.A.11`이 바뀌면 결제 실패 사용자 여정과 API 흐름을 확인한다.
- OpenAPI나 이벤트 계약이 바뀌면 producer, consumer, E2E collection을 함께 변경한다.
- 목표 설계와 현재 구현이 다르면 실행 기록과 인수인계 문서에 차이를 남긴다.

## 5. 현재 구조 공백

blueprint의 `50-service-design`에는 구매의 catalog, order, payment, notification Context 상세 설계가 아직 없다. 새 Context를 추가하기 전까지 scenarios는 현재 구현 설명을 담당하지만 장기 목표 서비스 설계를 대신하지 않는다.
