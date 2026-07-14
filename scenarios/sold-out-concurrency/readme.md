# 품절/동시성 시나리오

작성일: 2026-07-07

이 폴더는 DropMong에서 여러 고객이 같은 드롭에 동시에 구매를 시도할 때 성공, 품절, 요청 제한 결과가 일관되게 나오는지 설계한다. 기준 문서는 `../../medikong/12-user-flows.md`의 "품절/동시성 시나리오 오너"와 `../../blueprint/00-requirements/REQ_A_01_limited_drop_commerce.md`의 재고 정합성 요구사항이다.

## 목표

판매 수량이 제한된 드롭에서 동시 주문이 몰려도 성공 예약 수가 판매 가능 수량을 넘지 않게 한다. 실패한 고객은 품절, 요청 제한, 재시도 안내 중 하나를 명확하게 받는다.

```text
드롭 오픈
-> 여러 고객 동시 주문
-> admission 또는 order transaction
-> 일부 예약 성공
-> 나머지 품절/요청 제한
-> 결과 확인
```

## 문서

| 문서 | 내용 |
| --- | --- |
| `00-detailed-design.md` | 사용자 흐름, API, 데이터, transaction, 테스트, 인프라 상세 설계 |
| `01-user-journey.md` | 동시 주문에서 성공·품절 결과를 받는 사용자 흐름 |
| `02-api-flow.md` | `POST /orders`와 병렬 요청의 API 판정 경계 |
| `03-state-event-flow.md` | advisory lock, 활성 예약 합계, 상태와 멱등성 |
| `04-service-implementation-plan.md` | order-service 중심의 현재 구현과 후속 작업 |
| `05-test-scenarios.md` | PostgreSQL 동시성, 병렬 HTTP, E2E 수용 테스트 |
| `test-execution-record.md` | 현재 구현 상태와 실제로 실행한 단위 테스트, E2E 테스트, 추가해야 할 통합 테스트 기록 |

## 현재 완료 범위

- 완료: PostgreSQL advisory transaction lock, 병렬 주문, 초과 판매 방지, 품절 409, 주문 REST 멱등성
- 미완료: 실제 admission control, 장시간 spike p95/p99, 다중 상품 주문, 운영 SLO

목표 설계의 `inventory_buckets` 조건부 update와 현재 구현의 활성 주문 예약 합계 방식은 다르다. 실제 동작은 `test-execution-record.md`와 `../_shared/03-purchase-development-handoff.md`를 기준으로 확인한다.

## 관련 시나리오

| 시나리오 | 관계 |
| --- | --- |
| 정상 구매 | 성공한 고객은 정상 구매의 결제 승인 흐름으로 이어진다. |
| 결제 실패 | 결제 실패나 TTL 만료가 발생하면 예약 재고를 release해야 한다. |
