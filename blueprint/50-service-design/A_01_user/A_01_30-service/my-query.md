---
id: SD.A.0130.MY
title: 마이 화면의 본인 프로필 조회 경계
type: service-design-service
status: draft
tags: [service-design, user, my, frontend, ingress]
source: local
created: 2026-07-10
updated: 2026-07-13
service_design: SD.A.01
domain_model: SD.A.0110
---

# 마이 화면의 본인 프로필 조회 경계

## 결정

마이 화면은 컴포넌트별로 소유 서비스를 호출한다. 사용자 서비스는 기존 `QRY.A.01-01 GetOwnProfile`과 `API.A.01-02`로 본인 프로필만 제공한다. 마이 전용 Query와 통합 응답은 만들지 않는다.

## User 응답

- `userId`
- `accountStatus`
- `nickname`
- `introduction`
- nullable `profileMediaAssetId`
- `userVersion`
- `updatedAt`

이메일, 휴대폰, role/permission, 주문, 쿠폰, 포인트, 등급, 찜과 알림은 포함하지 않는다.

## 프론트엔드 책임

1. 프로필 컴포넌트가 Ingress를 통해 `API.A.01-02`를 호출한다.
2. 응답의 asset ID가 있으면 이미지 컴포넌트가 Ingress를 통해 Media에 signed URL을 요청한다.
3. 주문·쿠폰·포인트 컴포넌트는 각각 소유 서비스를 호출한다.
4. 컴포넌트별 로딩·성공·오류 상태를 독립적으로 표시한다.

Media 실패는 User 조회를 실패로 바꾸지 않는다. 프론트엔드는 기본 이미지를 표시하고 나머지 프로필 정보를 유지한다.

## 보안

- User와 각 서비스는 Session에서 얻은 사용자 Principal을 직접 검증한다.
- Ingress는 외부에서 들어온 내부용 헤더를 제거하며 응답을 합치지 않는다.
- signed URL은 표시 시점에만 만들고 저장하지 않는다.
- 본인 프로필 응답은 private cache 정책을 사용한다.

## 관련 시퀀스

- [마이 화면 컴포넌트 독립 조회](../A_01_50-sequence/SCN_A_01_05_load_my_components.md)
