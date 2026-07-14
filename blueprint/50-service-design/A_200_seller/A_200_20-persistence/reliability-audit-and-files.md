---
id: SD.A.20020.RELIABILITY
title: 판매자 멱등·outbox·감사·export 파일 수명주기
type: service-design-persistence
status: draft
tags: [service-design, seller, idempotency, outbox, audit, export]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
persistence: SD.A.20020
---

# 판매자 멱등·outbox·감사·export 파일 수명주기

공통 원장이라는 표현은 공통 schema 규칙을 뜻한다. 각 table은 해당 Aggregate 소유 서비스의 데이터베이스에 있고, 서비스 간에 공유하지 않는다. 소유 서비스가 미확정인 Aggregate의 table은 구현하지 않는다.

## 공통 원장

| table 후보 | 필수 키 | 보존 내용 |
| --- | --- | --- |
| `seller_idempotency_records` | `(api_id, principal_id, seller_id, idempotency_key)` | request HMAC, 상태, HTTP 결과 ref, 만료 정책 ref |
| `seller_outbox_events` | `event_id`, `(aggregate_id, aggregate_version, event_type)` | envelope, payload schema version, publish 상태 |
| `seller_inbox_events` | `(consumer_name, event_id)` | payload hash, source version, 처리 결과·실패 ref |
| `seller_audit_entries` | `audit_id`, `(seller_id, occurred_at)` | actor 또는 worker ref, action, target ref, before/after hash, reason ref, request·trace ID |
| `seller_state_transitions` | `(aggregate_id, to_version)` | from/to state, command ID, event ID |
| `seller_export_downloads` | `download_id`, `(export_id, occurred_at)` | membership ref, purpose, result, request ID |

request body 원문, proof, seller scope token, cookie, Authorization, signed URL, 주문 개인정보와 evidence 파일 원문은 이 원장에 넣지 않는다.

## 멱등과 동시성

- 범위는 `API ID + 인증 principal + seller_id + Idempotency-Key`다. 같은 key의 다른 canonical request HMAC은 `SELLER_IDEMPOTENCY_CONFLICT`다.
- 성공 replay는 최초 상태 code와 비민감 응답을 반환한다. 현재 상세가 필요하면 Query를 다시 호출한다.
- `If-Match`/expected version 불일치는 `SELLER_VERSION_CONFLICT`이며 서버가 자동 병합하지 않는다.
- 멱등 record만 저장되고 Aggregate가 실패하거나 그 반대인 상태가 생기지 않도록 같은 트랜잭션을 사용한다.

## export 파일 수명주기

1. 요청 트랜잭션에서 `OrderExport.REQUESTED`, idempotency, audit, `EVT.A.200-12` outbox를 저장한다.
2. Worker는 seller scope와 요청 snapshot을 다시 확인하고 `RM.A.200-07`의 허용 필드만 private object로 쓴다.
3. object checksum·row count·watermark 저장에 성공한 뒤 `CMD.A.200-13`으로 `READY`를 확정한다. 업로드만 성공한 orphan object는 수거 대상이다.
4. 다운로드 시마다 현재 membership·permission·강한 재인증과 expiry를 확인한 뒤 짧은 수명의 signed URL을 즉석 발급한다.
5. 만료 시 `CMD.A.200-14`로 접근을 먼저 차단하고 object 삭제를 재시도한다. 삭제 실패를 파일 사용 가능 상태로 되돌리지 않는다.

보존 기간·signed URL TTL·재인증 freshness 수치는 정책 원천이 정할 때까지 이 문서에서 정하지 않는다.
