---
id: SD.A.20030.SECURITY
title: 판매자 보안·오류·가용성 계약
type: service-design-service
status: draft
tags: [service-design, seller, security, errors, availability, ingress]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
service: SD.A.20030
---

# 판매자 보안·오류·가용성 계약

## 요청 신뢰 경계

1. 판매자 브라우저의 페이지와 API 요청은 Kubernetes Ingress를 통과한다. 목표 구조에는 Seller BFF가 없다.
2. Ingress는 외부 클라이언트가 보낸 `X-User-*`, `X-Seller-*`, role과 permission 값을 제거하고, Auth가 검증한 `user_id`, Session ref와 인증 정보를 신뢰할 수 있는 내부 형식으로 전달한다.
3. Ingress가 전달한 인증 context는 seller 업무 권한이 아니다. 수신 서비스는 현재 seller membership, permission version과 리소스의 seller 소유권을 다시 확인한다.
4. seller membership 원장 서비스가 아직 정해지지 않았으므로 `API.A.200-01~08`을 포함한 seller 보호 API는 현재 공개할 수 없다. Ingress에 고정 role이나 임시 seller ID를 넣어 우회하지 않는다.
5. `catalog-service`, `order-service`, `coupon-service`, `notification-service`는 자기 리소스의 seller scope를 다시 확인한다. 서명이나 Gateway 통과만으로 접근을 허용하지 않는다.

인증 context의 서명 방식, key rotation, audience와 최대 TTL은 배포 보안 정책에서 정한다. 미확정 숫자를 운영 기본값으로 만들지 않는다. 현재 OpenAPI의 `WorkloadBearerAuth`와 `X-Seller-Scope`는 BFF 전제의 내부 초안이므로 Ingress-facing 계약으로 확정하지 않는다.

## 강한 재인증

| purpose | 사용 작업 | Auth 계약 |
| --- | --- | --- |
| `seller_order_export` | `CMD.A.200-12`, 다운로드 URL 발급 | `API.A.300-17` proof, Session·seller·operation binding |
| `seller_member_manage` | `CMD.A.200-03~05` | 동일 |
| `seller_account_update` | `CMD.A.200-01`의 민감 정보 변경 | 동일 |
| `seller_partnership_respond` | 제휴 제안 응답 | 외부 계약 대기가 해소된 뒤 사용 |

재인증 proof는 membership이나 permission을 부여하지 않는다. 브라우저는 Auth가 발급한 목적 한정 증명을 업무 요청에 제출하고, 실제 소유 서비스는 현재 권한과 resource version을 다시 확인한다. 재인증 완료만으로 원래 Command를 자동 실행하지 않는다.

## 오류 계약

소유 서비스의 prefix가 정해질 때까지 아래 code는 논리 분류다. 실제 코드명은 각 서비스 OpenAPI에 편입할 때 해당 서비스 namespace로 확정한다.

| Code | HTTP | 의미 |
| --- | --- | --- |
| `SELLER_INPUT_INVALID` | 400 | schema·header·cursor가 유효하지 않음 |
| `SELLER_AUTHENTICATION_REQUIRED` | 401 | 검증된 인증 context 없음 |
| `SELLER_FORBIDDEN` | 403 | 현재 membership·permission·강한 재인증 불충족 |
| `SELLER_NOT_FOUND` | 404 | 없음과 다른 seller 소유 대상을 같은 응답으로 숨김 |
| `SELLER_IDEMPOTENCY_CONFLICT` | 409 | 같은 key의 다른 canonical request |
| `SELLER_VERSION_CONFLICT` | 409 | ETag·expected Aggregate·permission version 불일치 |
| `SELLER_STATE_CONFLICT` | 409 | 현재 상태에서 전이 불가 |
| `SELLER_BUSINESS_RULE_REJECTED` | 422 | 완결성·수량·마스킹·정책 규칙 불충족 |
| `SELLER_DEPENDENCY_UNAVAILABLE` | 503 | 필수 외부 원천·저장소 사용 불가 |
| `SELLER_PROJECTION_UNAVAILABLE` | 503 | 요청한 필수 Read Model section 없음 |

`503`은 빈 성공 응답으로 바꾸지 않는다. 소유 서비스의 Problem Details에 `dependency`, `retryable`, `retryAfter`와 안전한 `lastSuccessfulAsOf`를 제공할 수 있지만, 필드와 값은 해당 서비스 계약에서 확정한다.

## stale·partial

- 필수 권한·소유 원장은 stale 허용 대상이 아니다. 확인할 수 없으면 Command와 민감 Query를 실패시킨다.
- 대시보드·분석처럼 여러 서비스의 결과가 필요한 Query는 소유 서비스와 조회 모델이 생기기 전까지 API를 제공하지 않는다.
- 조회 모델이 확정된 뒤에만 정상 section과 `partial=true`, `unavailableSections`를 함께 반환할 수 있다.
- stale snapshot의 `asOf`를 현재 시각으로 바꾸거나, 다른 seller 데이터와 마스킹 전 개인정보를 대체값으로 사용하지 않는다.
