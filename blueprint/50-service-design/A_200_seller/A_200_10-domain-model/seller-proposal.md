---
id: SD.A.20010.PROPOSAL
title: 판매자 상품·드롭 제안 도메인 모델
type: service-design-domain-model
status: draft
tags: [service-design, seller, product, drop-proposal, review]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
domain_model: SD.A.20010
---

# 판매자 상품·드롭 제안 도메인 모델

## 모델과 소유권

| 모델 | 종류 | 소유 데이터 |
| --- | --- | --- |
| `SellerProduct` | Aggregate | seller 상품 원본, media ref, 설명, 가격 제안, option·공급 선언, version |
| `DropProposal` | Aggregate | product ref, 일정·수량·구매 제한·배송·반품 조건, 제출 version, review 상태, 승인 snapshot ref |
| `DropReview` | Entity | review request ID, 제출 proposal version, 외부 decision ID, decision·reason ref·decidedAt |
| `ProposalChangeRequest` | Entity | 승인 version, 변경 전후 canonical diff, 사유 ref, 상태 |

공개 Catalog와 Drop Context는 승인된 snapshot을 소유하거나 투영한다. 승인 전 seller 원본을 직접 읽지 않는다.

## 불변조건

- `SellerProduct.seller_id`와 `DropProposal.seller_id`가 현재 scope와 같아야 한다.
- option별 판매 수량 합계는 proposal 총 판매 수량과 같고 음수·부동소수 수량을 허용하지 않는다.
- 검수 제출은 상품·옵션·공급·배송·반품 필수 값과 media 검증이 끝난 단일 proposal version을 고정한다.
- `SUBMITTED`, `UNDER_REVIEW`, `APPROVED` version은 직접 수정하지 않는다. 반려 후 새 draft version 또는 승인 후 `ProposalChangeRequest`를 만든다.
- 외부 review result는 review request ID와 정확한 proposal version이 맞아야 한다. 늦거나 순서가 뒤집힌 결과는 성공 처리하지 않고 conflict 원장에 남긴다.
- `APPROVED`와 decision version이 일치할 때만 `CMD.A.200-11`이 승인 snapshot을 전달한다. handoff는 동일 proposal version당 한 번의 업무 결과만 가진다.

## 상태 전이

| 시작 | Command | 결과 | Event |
| --- | --- | --- | --- |
| 없음·`DRAFT` | `CMD.A.200-06` | `SellerProduct` 새 version | `EVT.A.200-06` |
| 없음·`DRAFT`·`REJECTED` | `CMD.A.200-07` | `DropProposal.DRAFT` 새 version | `EVT.A.200-07` |
| `DRAFT` | `CMD.A.200-08` | `SUBMITTED`와 고정 submit version | `EVT.A.200-08` |
| `SUBMITTED`·`UNDER_REVIEW` | `CMD.A.200-09` | `APPROVED`, `REJECTED`, `ON_HOLD` | `EVT.A.200-09` |
| `APPROVED` | `CMD.A.200-10` | `CHANGE_REQUESTED` | `EVT.A.200-10` |
| `APPROVED` | `CMD.A.200-11` | `HANDED_OFF`와 external handoff ref | `EVT.A.200-11` |

검수 decision taxonomy·SLA, 승인 후 원본 이관과 공개 Drop 응답 schema는 소유 Context 계약이 확정될 때까지 미확정 상태로 둔다.
