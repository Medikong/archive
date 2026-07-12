---
id: API.A.19-18
title: 쿠폰 읽기 전용 안내 적용 API
type: service-design-api-endpoint
status: draft
tags: [service-design, coupon, api, operations, notice]
source: local
created: 2026-07-11
updated: 2026-07-12
service_design: SD.A.19
api_design: SD.A.1940
domain_model: SD.A.1910
persistence: SD.A.1920
service: SD.A.1930
---

# API.A.19-18 쿠폰 읽기 전용 안내 적용

## 기본 정보

| 항목 | 값 |
| --- | --- |
| Method / Path | `PUT /api/v1/internal/coupon-operational-controls/{controlId}/read-only-notice` |
| operationId | `applyCouponReadOnlyNotice` |
| 역할 | 운영 중지와 같은 범위의 안내 문구·적용 시각·활성 상태를 저장한다. |
| API 유형 | Command |
| 인증·권한 | 운영 workload, approvalRef |
| 노출 범위 | internal |
| 멱등성 | `Idempotency-Key`와 `expectedVersion` 필수 |
| 캐시 | `no-store` |
| 호환성 | notice schema version 1 |

## HTTP 계약 원장

- [OpenAPI](openapi/openapi.yaml)
- [Path Item](openapi/paths/API_A_19_18_apply_coupon_read_only_notice.yaml)

## 연관 문서

| 구분 | 연결 |
| --- | --- |
| 요구사항·UC | `REQ.A.02.FR-016`, `UC.A.19-15` |
| BC | `CMD.A.19-31`, `EVT.A.19-30`, `RM.A.19-09` |
| 서비스 | `ApplyCouponReadOnlyNoticeHandler` |

## 책임과 경계

- 기존 `CouponOperationalControl`과 같은 scope에 표시 안내를 연결한다.
- 안내가 발급·사용 중지 상태 자체를 대신하지 않는다.
- 다국어 번역과 채널 발송은 이 Aggregate의 책임이 아니다.

## 보안과 개인정보

- control 접근 권한, approvalRef와 expected version을 확인한다.
- 안내 문구에 사용자·주문·인시던트 식별자나 내부 장애 원문을 허용하지 않는다.

## 처리 규칙

1. control scope와 현재 version을 확인한다.
2. 문구 길이·안전 정책, 적용 시각과 활성 상태를 검증한다.
3. 안내와 outbox를 저장하고 `RM.A.19-09` 투영을 갱신한다.

## 상태 변경과 트랜잭션

- `CouponOperationalControl`, 안내 원장과 outbox를 원자적으로 저장한다.
- 쿠폰함 Query는 투영된 활성 안내만 읽는다.

## 멱등성과 동시성

- 범위는 `control_id + notice_version + Idempotency-Key`다.
- expected version으로 병렬 안내 수정의 lost update를 막는다.
- 같은 key replay는 기존 notice version을 반환한다.

## 예외와 복구 규칙

| 업무 조건 | 공개 원칙 | 복구 |
| --- | --- | --- |
| control 없음·권한 없음 | 범위 상세를 숨김 | control과 권한을 확인한다. |
| 문구 정책 위반 | 위반 유형만 반환 | 안전한 문구로 수정한다. |
| 투영 지연 | 저장 결과를 되돌리지 않음 | outbox를 재처리한다. |

## 도메인과 서비스 매핑

| 계층 | 매핑 |
| --- | --- |
| Handler | `ApplyCouponReadOnlyNoticeHandler` |
| Aggregate | `CouponOperationalControl` |
| Repository | `CouponOperationalControlRepository` |
| Read Model | `CouponReadOnlyNotice` |
| Event | `EVT.A.19-30` |

## 관측성과 운영

- notice version, 활성 여부, 적용 지연을 audit하되 문구 원문을 metric에 넣지 않는다.

## 검증 항목

- 중지 scope와 다른 범위에 안내가 노출되지 않는다.
- version 충돌이 기존 안내를 덮어쓰지 않는다.
- 비활성화 Event 뒤 쿠폰함에서 안내가 제거된다.

## 연관 시퀀스

- `API.A.19-17`의 control 생성 뒤 호출한다.

## 호환성과 변경 정책

- 표시 metadata는 선택 필드로 추가하고 문구의 기존 의미를 재사용하지 않는다.

## 결정 반영

- `HOTSPOT.A.19-05`: 안내 적용은 승인된 운영 작업 참조와 버전이 있는 위험 기반 승인 정책을 따른다.
