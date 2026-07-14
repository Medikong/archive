---
id: SD.A.20010.MANAGEMENT
title: 판매자 계정·스토어·membership 도메인 모델
type: service-design-domain-model
status: draft
tags: [service-design, seller, account, store, membership, role]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.200
domain_model: SD.A.20010
---

# 판매자 계정·스토어·membership 도메인 모델

## 소유권

| 모델 | 종류 | 소유 데이터 |
| --- | --- | --- |
| `SellerAccount` | Aggregate | `seller_id`, seller type, verification snapshot, 책임 고지, 고객 안내 연락처, account status, version |
| `StoreProfile` | Aggregate | store name, logo·cover asset ref, introduction, buyer-facing notice, version |
| `SellerTeam` | Aggregate | seller별 membership·role 집합, owner 불변조건, team version과 permission version |
| `SellerMembership` | Entity | membership ID, seller ID, `user_id`, role ID, status, membership version, permission version |
| `SellerRole` | Entity/정책 | role ID, 표시명, permission set, system/custom 구분, version |

Auth의 이메일·휴대폰·credential·Session은 저장하지 않는다. 사용자 연결 키는 Auth가 인증한 `user_id`뿐이다.

## 불변조건

- 모든 Aggregate ID와 unique key는 `seller_id` 범위를 포함한다. 다른 seller의 동일 resource ID는 존재 여부를 숨긴 같은 `404`로 처리한다.
- `SellerAccount`의 책임 고지 필수 항목이 완결되지 않으면 verified·active 전이를 허용하지 않는다.
- `StoreProfile`의 media는 검증된 asset ref만 연결하며 signed URL을 원장에 저장하지 않는다.
- active seller에는 비활성화되지 않은 owner 역할 membership이 적어도 하나 있어야 한다. 마지막 owner 비활성화·권한 제거는 거절한다.
- role 변경과 membership 비활성화는 `permission_version`을 증가시키고 다음 요청부터 이전 signed scope를 무효화한다.
- 초대는 기존 `user_id`에 대한 제안이며 이메일 주소를 membership 원장이나 Event에 넣지 않는다.
- 다중 seller 선택 정책이 확정되기 전까지 API는 항상 target `seller_id`를 명시하며 자동 seller 선택을 하지 않는다.

## 상태 전이

| 모델 | 시작 | Command | 결과 | 거절 조건 |
| --- | --- | --- | --- | --- |
| `SellerAccount` | `DRAFT`, `ACTIVE`, `RESTRICTED` | `CMD.A.200-01` | 동일 상태의 새 version 또는 검증 정책이 허용한 상태 | 책임 고지 누락, stale version, 승인 snapshot 불일치 |
| `StoreProfile` | `DRAFT`, `PUBLISHED` | `CMD.A.200-02` | 새 profile version | media 미검증, 금지 문구, stale version |
| `SellerTeam`/`SellerMembership` | 없음 | `CMD.A.200-03` | `INVITED` membership | owner 아님, 중복 active/invited membership |
| `SellerTeam`/`SellerMembership` | `INVITED`, `ACTIVE` | `CMD.A.200-04` | `ACTIVE`와 새 role/version | 마지막 owner 제거, 권한 상승 승인 누락, stale version |
| `SellerTeam`/`SellerMembership` | `INVITED`, `ACTIVE` | `CMD.A.200-05` | `INACTIVE` | 마지막 owner, 이미 inactive인 다른 canonical 요청 |

초대 수락 방법과 만료 기본값은 BC에 없으므로 별도 성공 계약을 만들지 않고 `GAP`으로 남긴다. 현재 API는 초대 생성·역할 변경·비활성화까지만 제공한다.

## Event

`EVT.A.200-01~05`는 seller ID, Aggregate ID/version, 변경 field set, actor ref와 발생 시각만 전달한다. role 전체 권한표나 사용자 프로필은 Event payload에 넣지 않는다.
