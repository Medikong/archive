---
id: SD.A.0740
title: Context 관심/랭킹 API 설계 인덱스
type: service-design-api
status: draft
tags: [service-design, interest, wishlist, ranking, api, openapi]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.07
bounded_context: BC.A.07
domain_model: SD.A.0710
persistence: SD.A.0720
service: SD.A.0730
---

# Context 관심/랭킹 API 설계 인덱스

## 역할

HTTP 계약은 OpenAPI로, 업무 처리·트랜잭션·멱등성·동시성 판단은 [api-endpoint-process.md](api-endpoint-process.md)로 나누어 관리한다.

## 연관 태그

🏷️ BC 참조: [BC.A.07](../../../40-event-storming-bounded-context/BC_A_07_interest_ranking.md) | 도메인 참조: [SD.A.0710](../A_07_10-domain-model/SD_A_0710_interest_domain_model.md) | 영속성 참조: [SD.A.0720](../A_07_20-persistence/persistence-design.md) | 서비스 참조: [SD.A.0730](../A_07_30-service/service-design.md)

## 원장 구분

| 원장 | 담당 내용 |
| --- | --- |
| `openapi/openapi.yaml` | 공개 API 진입 문서, 서버와 security scheme 등록 |
| `openapi/paths/*.yaml` | Method/Path, parameter, request body, response, HTTP 상태, 오류, 예시 |
| `openapi/components/*.yaml` | 재사용 schema, parameter, response header, response, security scheme |
| `api-endpoint-process.md` | 책임과 경계, 상태 변경, 트랜잭션, 멱등성, 동시성, 보안 근거 |

## API 목록

| API ID | operationId | Method / Path | UC | Command/Query | 노출 범위 |
| --- | --- | --- | --- | --- | --- |
| `API.A.07-01` | `addInterest` | `PUT /v1/users/me/interests/{dropId}` | [UC.A.07](../../../30-uc/UC_A_07_interest_ranking.md) | Command | public(로그인 필요) |
| `API.A.07-02` | `removeInterest` | `DELETE /v1/users/me/interests/{dropId}` | UC.A.07 | Command | public(로그인 필요) |
| `API.A.07-03` | `listMyInterests` | `GET /v1/users/me/interests` | UC.A.07 | Query | public(로그인 필요) |
| `API.A.07-04` | `recordDropView` | `POST /v1/drops/{dropId}/views` | UC.A.07 | Command | public(로그인 필요) |
| `API.A.07-05` | `listOpenRanking` | `GET /v1/rankings/drops/open` | UC.A.07 | Query | public |
| `API.A.07-06` | `listUpcomingRanking` | `GET /v1/rankings/drops/upcoming` | UC.A.07 | Query | public |
| `API.A.07-07` | `getDropInterestStats` | `GET /v1/operator/drops/{dropId}/interest-stats` | UC.A.07 | Query | operator(`role=operator` 전용, 2026-07-13 결정으로 브랜드 운영자는 1차 범위 밖) |

## 작성 규칙

- Request/Response 필드, JSON 예시, HTTP 상태와 오류 body는 OpenAPI에만 작성한다.
- OpenAPI에는 미확정 결정을 넣지 않는다. 미확정 사항은 `api-endpoint-process.md`의 "확인 필요"에 둔다.
- `x-api-id`, `x-use-case`, `x-service-design`로 추적성을 유지한다.
- 인증이 필요한 Endpoint(`API.A.07-01`, `-02`, `-03`, `-04`, `-07`)는 `Authorization: Bearer <JWT>`(`BearerAuth`) + Gateway가 검증 후 전달하는 `X-User-Id`/`X-User-Email`/`X-User-Role` 헤더를 요구한다 — `service` 레포 `contracts/jwt-conventions.md`, catalog-service/order-service 공식 계약(`contracts/services/*/openapi.yaml`)과 동일한 컨벤션(Python 구현 확정에 따라 2026-07-13 정정, `X-Principal`은 Go 서비스 전용이라 채택하지 않음). archive의 `A_300_auth` 설계 문서가 제안하는 Cookie+CSRF+세밀한 permission 방식은 아직 실제 코드에 구현돼 있지 않으므로 지금은 따르지 않는다.
- 단일 리소스 응답(`InterestResponse`, `ViewRecordResponse`, `DropInterestStatsResponse`)은 catalog-service의 `DropDetailResponse`, order-service의 `OrderResponse`와 동일하게 `{data: ...}`로 감싼다(Go의 coupon-service는 언랩된 응답을 쓰지만, 우리는 Python 공식 계약 컨벤션을 따른다).

## 검증

`archive` 저장소 루트에서 실행한다.

```bash
npx --yes @redocly/cli@2.38.0 lint \
  --config blueprint/50-service-design/A_07_interest_ranking/A_07_40-api/openapi/redocly.yaml \
  blueprint/50-service-design/A_07_interest_ranking/A_07_40-api/openapi/openapi.yaml
```

## 확인 필요

- `API.A.07-07`의 경로 접두사 `/operator/...`는 `contracts/jwt-conventions.md` 문서 규칙을 따른 것이지만, 실제 코드의 유일한 선례(`backoffice-service`)는 같은 role 체크를 하면서 `/admin/...`을 쓴다 — 팀과 접두사를 확정할지.
- 브랜드 운영자(`ACTOR.A.07-03`)의 자사 드롭 소유권 검증을 어느 Context(드롭 관리? 판매자 서비스?)에서 가져올지 정해지면 `API.A.07-07`의 스코프를 확장할지 별도 API로 분리할지 재검토한다.
- 목록 API(`-03`, `-05`, `-06`)에 `limit`/`cursor` pagination을 추가했다(`contracts/common/components.yaml`의 공용 parameter 재사용, 2026-07-13). 실제 커서 구현(어떤 컬럼 기준 정렬·인코딩)은 `SD.A.0720`/`SD.A.0730`에서 채울 것.
- Command Endpoint(`-01`, `-02`, `-04`)에 다른 서비스처럼 `Idempotency-Key` 헤더를 요구할지 — `-04`에는 이미 선택적으로 추가해둠(coupon-service의 `/v1/coupons/issue` 패턴 참고), `-01`/`-02`는 PUT/DELETE 자연 멱등성으로 충분하다고 판단해 생략함.
