---
id: SD.A.0740.01
title: Context 관심/랭킹 Endpoint 설계 노트
type: service-design-api-endpoint
status: draft
tags: [service-design, interest, wishlist, ranking, api, endpoint]
source: local
created: 2026-07-13
updated: 2026-07-13
service_design: SD.A.07
api_design: SD.A.0740
domain_model: SD.A.0710
persistence: SD.A.0720
service: SD.A.0730
---

# Context 관심/랭킹 Endpoint 설계 노트

## 공통 원칙

- 모든 Command Endpoint(`-01`, `-02`, `-04`)는 로그인 사용자만 대상이다(`POLICY.A.07-01`). 비로그인 요청은 `401`이다.
- Request/Response 필드, JSON 예시, HTTP 상태와 오류 body는 [openapi/openapi.yaml](openapi/openapi.yaml)을 기준으로 한다. 이 문서는 업무 처리·트랜잭션·멱등성 판단만 다룬다.
- 내부 식별자와 상세 실패 원인은 클라이언트에 노출하지 않는다. 오류는 업무 조건만 공개한다.

## API.A.07-01 찜 추가

| 항목 | 값 |
| --- | --- |
| Method / Path | `PUT /v1/users/me/interests/{dropId}` |
| operationId | `addInterest` |
| 역할 | 로그인 사용자를 특정 드롭에 찜한 상태로 만든다. |
| 유형 | Command |
| 인증 | `Authorization: Bearer <JWT>` + `X-User-Id`/`X-User-Role`(auth-service 발급, Gateway가 검증 후 전달) |
| 멱등성 | 필수. `(user_id, drop_id)` 자체가 멱등 키 — 같은 요청 반복은 같은 결과. |

- 책임과 경계: 보장하는 것은 `Interest.status = active`. 랭킹 카운터 반영은 이 요청 안에서 보장하지 않는다(비동기, `RULE.A.07-03`).
- 처리 규칙: 로그인 게이트 → `InterestRepository.findByUserAndDrop` → 이미 `active`면 상태 변경 없이 `200` → `inactive`거나 신규면 `active`로 전이 후 `201`/`200` → 찜 추가됨 이벤트 발행.
- 상태 변경과 트랜잭션: 시작 상태 `inactive`/없음 → 종료 상태 `active`. `interests` 단일 행만 같은 트랜잭션에 저장한다.
- 멱등성과 동시성: 같은 `(user_id, drop_id)` 재요청은 같은 최종 상태를 반환한다(PUT의 자연적 멱등성). 동시 요청은 `version` 낙관적 잠금으로 방어하고 충돌 시 재조회 후 재시도.
- 예외: 드롭이 존재하지 않으면 `404`, 인증 실패는 `401`.
- 도메인 매핑: Handler → `InterestCommandHandler.add` / Aggregate → `Interest`(`AGG.A.07-01`) / Repository → `InterestRepository` / Event → `EVT.A.07-01`.

## API.A.07-02 찜 해제

| 항목 | 값 |
| --- | --- |
| Method / Path | `DELETE /v1/users/me/interests/{dropId}` |
| operationId | `removeInterest` |
| 역할 | 로그인 사용자의 찜 상태를 해제한다. |
| 유형 | Command |
| 인증 | `Authorization: Bearer <JWT>` + `X-User-Id`/`X-User-Role` |
| 멱등성 | 필수. DELETE 자연적 멱등성 — 이미 없는 찜을 다시 해제해도 `active` 상태가 아니게만 보장. |

- 책임과 경계: `Interest.status = inactive`로 전이. `Interest` 레코드 자체를 삭제하지 않는다(찜/해제 반복 이력을 같은 행에서 관리).
- 처리 규칙: 로그인 게이트 → 조회 → 이미 `inactive`거나 레코드 없음이면 `204`(변경 없음) → `active`면 `inactive`로 전이 후 `204` → 찜 해제됨 이벤트 발행.
- 상태 변경과 트랜잭션: `active → inactive`. `interests` 단일 행.
- 멱등성과 동시성: `API.A.07-01`과 동일한 `version` 낙관적 잠금.
- 예외: 인증 실패 `401`. 존재하지 않는 드롭이어도 찜 자체가 없으므로 `204`로 처리(정보 노출 최소화).
- 도메인 매핑: Handler → `InterestCommandHandler.remove` / Aggregate → `Interest` / Event → `EVT.A.07-02`.

## API.A.07-03 찜 목록 조회

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /v1/users/me/interests` |
| operationId | `listMyInterests` |
| 역할 | 로그인 사용자가 찜한 드롭 목록을 반환한다(`RM.A.07-01`). |
| 유형 | Query |
| 캐시 | `no-store`(즉시 정합성 요구) |

- 책임과 경계: `interests WHERE user_id = ? AND status = 'active'`만 반환한다. 드롭 상세 정보(가격, 이미지 등)는 이 API의 책임이 아니며 클라이언트가 catalog-service 조회와 조합한다.
- 처리 규칙: 로그인 게이트 → `InterestRepository.listActiveByUser` → `drop_id` 목록 반환.
- 상태 변경: 없음(Query).
- 도메인 매핑: Repository → `InterestRepository.listActiveByUser` / Read Model → `RM.A.07-01`.
- `limit`/`cursor` 공용 pagination parameter를 사용한다(2026-07-13 추가).

## API.A.07-04 드롭 조회 기록

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /v1/drops/{dropId}/views` |
| operationId | `recordDropView` |
| 역할 | 드롭 상세 화면 진입을 관심 신호로 기록한다(`RULE.A.07-04`, catalog-service를 거치지 않는 직접 호출). |
| 유형 | Command |
| 멱등성 | 5분 dedup 윈도우 안에서 멱등(`POLICY.A.07-02`). |

- 책임과 경계: 로그인 사용자의 조회만 집계한다(`POLICY.A.07-01`). 비로그인 조회는 이 API를 호출하지 않는 것을 프론트 계약으로 하며, 서버도 인증 실패 시 집계하지 않는다.
- 처리 규칙: 로그인 게이트 → Redis `SETNX view:{user_id}:{drop_id}` `EX 300` → 실패(이미 존재)면 카운터 변경 없이 `200` → 성공이면 `DropInterestCounterRepository.applyView` 호출 후 `200`.
- 상태 변경과 트랜잭션: `drop_interest_counters.view_count_total`, `upcoming_count` 증가. dedup 키(Redis)는 DB 트랜잭션 밖의 선행 단계.
- 멱등성과 동시성: dedup 키가 곧 멱등 범위(사용자·드롭·5분). Redis 성공 후 DB 갱신 실패 시 처리는 확인 필요.
- 예외: 인증 실패 `401`. dedup에 걸린 경우도 오류가 아니라 `200`(클라이언트는 성공/실패를 구분할 필요 없음).
- 도메인 매핑: Handler → `ViewRecordingHandler` / Aggregate → `DropInterestCounter`(`AGG.A.07-02`) / Event → `EVT.A.07-03`.

## API.A.07-05 오픈 후 인기 랭킹 조회

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /v1/rankings/drops/open` |
| operationId | `listOpenRanking` |
| 역할 | `OPEN` 상태 드롭을 `sell_through_score` 내림차순으로 반환한다(`RM.A.07-02`). |
| 유형 | Query |
| 인증 | 불필요(비로그인 열람 가능) |
| 캐시 | 짧은 TTL 캐시 검토(확인 필요, [SD.A.0720](../A_07_20-persistence/persistence-design.md)) |

- 책임과 경계: `drop_interest_counters WHERE drop_phase = 'OPEN' ORDER BY sell_through_score DESC`. 드롭 상세 원본은 제공하지 않는다(`drop_id`만).
- 상태 변경: 없음.
- 도메인 매핑: Repository → `DropInterestCounterRepository.listByPhaseOrderByScore`.
- `limit`/`cursor` 공용 pagination parameter를 사용한다(2026-07-13 추가).

## API.A.07-06 오픈 예정 랭킹 조회

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /v1/rankings/drops/upcoming` |
| operationId | `listUpcomingRanking` |
| 역할 | `SCHEDULED` 상태 드롭을 `upcoming_count` 내림차순으로 반환한다(`RM.A.07-03`). |
| 유형 | Query |
| 인증 | 불필요 |

- 책임과 경계: `drop_interest_counters WHERE drop_phase = 'SCHEDULED' ORDER BY upcoming_count DESC`.
- 도메인 매핑: Repository → `DropInterestCounterRepository.listByPhaseOrderByUpcoming`.
- `API.A.07-05`와 처리 구조가 동일하므로 같은 확인 필요 사항을 공유한다.

## API.A.07-07 드롭 관심도 통계 조회

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /v1/operator/drops/{dropId}/interest-stats` |
| operationId | `getDropInterestStats` |
| 역할 | DropMong 운영자가 드롭별 찜·조회·점수 통계를 확인한다(`RM.A.07-04`). |
| 유형 | Query |
| 인증 | `Authorization: Bearer <JWT>` + `X-User-Id`/`X-User-Role: operator` (실제 코드 컨벤션은 `contracts/jwt-conventions.md` 참고, role enum은 `CUSTOMER`/`OPERATOR`/`ADMIN`) |
| 노출 범위 | operator (`role=operator`, 2026-07-13 결정으로 브랜드 운영자는 1차 범위 밖) |

- 책임과 경계: `drop_interest_counters`의 `upcoming_count`, `view_count_total`, `sell_through_score`를 드롭 단위로 노출.
- 1차 스코프 결정(2026-07-13): 브랜드 운영자(`ACTOR.A.07-03`)의 자사 드롭 소유권 검증 메커니즘이 아직 없어 이 Endpoint는 `operator` role로만 제한한다. 브랜드 운영자 접근은 소유권 검증 방식이 정해진 뒤 별도 스코프 확장 또는 별도 API로 다룬다.
- 경로는 `service` 레포 `contracts/jwt-conventions.md`의 "드롭 운영자 API는 `/operator/...` prefix" 규칙을 따랐다. 다만 실제 코드의 유일한 선례인 `backoffice-service`는 같은 role 체크(`HasRole("operator")`)를 쓰면서도 경로는 `/v1/admin/...`을 쓰고 있어 문서-코드 불일치가 있다(확인 필요, 우리는 문서 규칙을 따름).
- 처리 규칙: `X-User-Id`/`X-User-Role` 확인(FastAPI `Header` 의존성, order-service의 `x_user_role: Annotated[UserRole, Header(alias="X-User-Role")]` 패턴과 동일) → role이 `OPERATOR`가 아니면 `403` → `DropInterestCounterRepository.findByDrop` 조회.
- 예외: `Authorization`/`X-User-*` 없음·무효 `401`, operator role 아님 `403`, 드롭 없음 `404`.
- 도메인 매핑: Repository → `DropInterestCounterRepository.findByDrop`.
- 확인 필요: 브랜드 운영자의 "자사 드롭" 판정 기준을 어느 Context에서 어떻게 조회할지.

## 관측성과 운영(공통)

- 로그: `user_id`(해시), `drop_id`, `operationId`, 처리 결과. 원문 조회수/찜 수는 로그가 아니라 메트릭으로 집계한다.
- Metric: Command 성공/실패율, dedup 적중률(`API.A.07-04`), 낙관적 잠금 충돌률(`API.A.07-01`, `-02`).
- Rate limit: `API.A.07-04`(조회 기록)는 핫키 상황에서 write가 몰릴 수 있어 개별 사용자 기준보다 드롭 기준 rate limit을 우선 검토한다(`POLICY.A.07-03`).

## 확인 필요(공통)

- `/v1/operator/...` vs `/v1/admin/...` 경로 접두사 불일치를 팀과 확정할지(위 참고).
- 커서(`cursor`) 인코딩 방식(어떤 컬럼 기준으로 정렬하고 어떻게 인코딩할지)은 `SD.A.0720`/`SD.A.0730`에서 구현 시 정한다.

## 2026-07-13 수정 이력

- 인증을 `WebSessionCookie`/`x-required-permissions`(archive의 `A_300_auth` 설계 문서 기준)에서 `X-Principal` 헤더 + `role` claim(`service` 레포 실제 코드 `contracts/jwt-conventions.md`, coupon-service/backoffice-service 기준)으로 정정.
- 경로를 `/api/v1/...`에서 `/v1/...`로 정정(실제 프로젝트는 `/api` 접두사를 쓰지 않음).
- 성공 응답의 `data`/`meta` 제네릭 래핑을 제거. 단일 리소스는 래핑 없이 그대로 반환(coupon-service의 `Policy`/`Coupon` 패턴), 목록은 `{data, pageInfo}`(catalog-service의 `DropListResponse` 패턴), 에러는 `{error: {code, message}, requestId, occurredAt}`로 정정.
- 목록 API(`-03`, `-05`, `-06`)에 `contracts/common/components.yaml`의 공용 `limit`/`cursor` parameter를 추가.

## 2026-07-13 수정 이력(2차, 구현 언어를 Python으로 확정한 뒤)

- 구현 언어를 Python(FastAPI)으로 확정하면서, 위 1차 수정에서 "coupon-service/backoffice-service 기준"으로 정한 `X-Principal` 헤더 인증이 실제로는 **Go 서비스 전용 컨벤션**이었음을 재확인했다. Python 서비스(catalog-service, order-service)는 `X-Principal`을 전혀 쓰지 않고 `Authorization: Bearer <JWT>`(`BearerAuth`) + Gateway가 전달하는 `X-User-Id`/`X-User-Email`/`X-User-Role` 헤더만 쓴다(`contracts/jwt-conventions.md`, `services/order-service/app/main.py`의 `Header(alias="X-User-Id")` 실제 구현으로 확인). 모든 Endpoint의 인증 방식을 이 컨벤션으로 다시 정정했다.
- 1차 수정에서 "단일 리소스는 래핑 없이 반환(coupon-service 기준)"으로 정한 것도 Go 전용 관례였다. Python 공식 계약(`contracts/services/catalog-service/openapi.yaml`의 `DropDetailResponse`, `contracts/services/order-service/openapi.yaml`의 `OrderResponse`)은 단일 리소스도 `{data: ...}`로 감싼다. `InterestResponse`/`ViewRecordResponse`/`DropInterestStatsResponse`를 새로 만들어 `Interest`/`ViewRecordResult`/`DropInterestStats`를 감싸도록 정정했다.
- catalog-service를 직접 확인한 결과 Kafka 연동이 전혀 없는 최소 스텁 상태임을 발견했다. `SD.A.0730`이 전제하는 `catalog.drop.updated` 이벤트 구독은 아직 실제로 검증할 수 없어, 첫 구현 범위를 `Interest` Aggregate(API.A.07-01/-02/-03)로 좁히고 `DropInterestCounter`(랭킹/이벤트) 관련 API(-04, -05, -06, -07)는 다음 단계로 미뤘다.
