---
id: SD.A.20010.OPERATIONS
title: 주문 자료·판매자 이슈 도메인 모델
type: service-design-domain-model
status: draft
tags: [service-design, seller, export, issue, privacy]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
domain_model: SD.A.20010
---

# 주문 자료·판매자 이슈 도메인 모델

## OrderExport

`OrderExport`는 export 요청·파일 ref·만료와 다운로드 감사를 소유한다. Order 원본이나 구매자 전체 개인정보를 소유하지 않는다.

| 필드 | 규칙 |
| --- | --- |
| `export_id`, `seller_id` | seller 범위의 opaque ID |
| `purpose` | allowlist 목적. 현재 확정 목적은 `fulfillment`이며 추가 목적은 정책 승인 필요 |
| `filter_snapshot` | 기간·상태·drop ref·format의 canonical snapshot |
| `data_watermark` | export에 반영한 `RM.A.200-07` 원천 watermark |
| `status` | `REQUESTED`, `GENERATING`, `READY`, `FAILED`, `EXPIRED` |
| `object_ref` | private object storage ref. signed URL은 저장하지 않음 |
| `expires_at` | 정책이 제공한 값. 이 설계에서 임의 기본 기간을 정하지 않음 |

`CMD.A.200-12`는 강한 재인증 purpose `seller_order_export`와 감사 가능한 접속 context를 요구한다. `CMD.A.200-13`은 요청 snapshot과 생성 결과의 hash·row count·watermark가 일치할 때만 `READY`로 만든다. `CMD.A.200-14` 뒤에는 새 signed URL을 발급하지 않으며 object 삭제 실패는 만료 상태를 되돌리지 않고 별도 삭제 작업으로 추적한다.

## SellerIssue

| 필드 | 규칙 |
| --- | --- |
| `issue_id`, `seller_id` | seller 범위 식별자 |
| `category`, `reason_code` | 허용 taxonomy ref. 자유 텍스트로 판정 의미를 대체하지 않음 |
| `subject_refs` | seller가 소유한 order·drop·proposal의 opaque ref |
| `description`, `evidence_refs` | 크기·형식·악성 파일 검사를 통과한 판매자 제출 내용 |
| `external_case_ref` | 플랫폼 운영 원장의 case ref |
| `status` | `OPEN`, `WAITING_SELLER`, `IN_REVIEW`, `RESOLVED`, `CLOSED` |

`CMD.A.200-15~16`은 seller 제출 원장과 outbox를 저장한다. `CMD.A.200-17`은 외부 결과를 복제하지 않고 case result ref와 seller에게 공개 가능한 요약만 반영한다. 페널티·정산 차감은 SellerIssue가 결정하지 않는다.

## 상태 전이

| 모델 | 시작 | Command | 결과 | Event |
| --- | --- | --- | --- | --- |
| `OrderExport` | 없음 | `CMD.A.200-12` | `REQUESTED` | `EVT.A.200-12` |
| `OrderExport` | `REQUESTED`, `GENERATING` | `CMD.A.200-13` | `READY` 또는 명시적 `FAILED` | `EVT.A.200-13` |
| `OrderExport` | `READY`, `FAILED` | `CMD.A.200-14` | `EXPIRED` | `EVT.A.200-14` |
| `SellerIssue` | 없음 | `CMD.A.200-15` | `OPEN` | `EVT.A.200-15` |
| `SellerIssue` | `OPEN`, `WAITING_SELLER`, `IN_REVIEW` | `CMD.A.200-16` | 새 timeline entry와 외부 case 전달 대기 | `EVT.A.200-16` |
| `SellerIssue` | `OPEN`, `WAITING_SELLER`, `IN_REVIEW` | `CMD.A.200-17` | `RESOLVED` 또는 `CLOSED`와 외부 result ref | `EVT.A.200-17` |

## 불변조건과 감사

- export와 issue의 모든 상태 전이는 actor/workload, reason ref, request ID, trace ID와 이전·새 version을 감사 원장에 남긴다.
- download마다 export ID, membership ID, 목적, 발생 시각과 결과를 append-only로 기록하고 signed URL·IP 원문은 저장하지 않는다.
- 다른 seller의 order·drop·issue ref와 존재하지 않는 ref는 같은 `404`로 처리한다.
- 파일 생성·삭제·외부 case 전달 실패를 `READY`·`RESOLVED` 성공으로 바꾸지 않는다.
