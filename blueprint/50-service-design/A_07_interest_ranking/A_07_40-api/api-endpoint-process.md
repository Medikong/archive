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

## API.A.07-04 드롭 조회 기록 (2026-07-14 재설계)

| 항목 | 값 |
| --- | --- |
| Method / Path | `POST /v1/drops/{dropId}/views` |
| operationId | `recordDropView` |
| 역할 | 드롭 상세 화면 진입을 관심 신호로 기록한다(`RULE.A.07-04`, catalog-service를 거치지 않는 직접 호출). |
| 유형 | Command |
| 멱등성 | 없음(원문 그대로 기록, 반복 호출 시 매번 새 행) — dedup은 이 API가 아니라 랭킹 집계 시점(`COUNT(DISTINCT user_id)`)에서 처리한다. |

- 책임과 경계: 로그인 사용자의 조회만 집계한다(`POLICY.A.07-01`). 비로그인 조회는 이 API를 호출하지 않는 것을 프론트 계약으로 하며, 서버도 인증 실패 시 집계하지 않는다.
- 처리 규칙(2026-07-14 단순화): 로그인 게이트 → `DropViewRepository.recordView(drop_id, user_id)`로 `drop_views`에 insert → `204`(본문 없음, `remove_interest`와 동일 패턴). Redis/dedup 단계와 `recorded` boolean이 더는 의미가 없어져 응답 스키마(`ViewRecordResponse`)도 제거했다.
- 상태 변경과 트랜잭션: `drop_views` insert 하나. 이 API 자체는 랭킹 값을 직접 갱신하지 않는다 — 랭킹은 3시간마다 별도 배치(`SD.A.0730`의 "실시간 조회 랭킹 배치 Worker")가 계산한다.
- 예외: 인증 실패 `401`.
- 도메인 매핑: Handler → `ViewRecordingHandler` / 모델 → `DropView`(신규, Aggregate 아님) / Event → `EVT.A.07-03`.

## API.A.07-05 오픈 후 인기 랭킹 조회 (보류)

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /v1/rankings/drops/open` |
| operationId | `listOpenRanking` |
| 역할 | `OPEN` 상태 드롭을 `sell_through_score` 내림차순으로 반환한다(`RM.A.07-02`). |
| 유형 | Query |
| 인증 | 불필요(비로그인 열람 가능) |
| 캐시 | 짧은 TTL 캐시 검토(확인 필요, [SD.A.0720](../A_07_20-persistence/persistence-design.md)) |

**2026-07-14: 이 API는 후속 스코프다.** order-service 연동 방식(동기 조회 vs `order.created` 비동기 구독 + raw 판매 속도로 공식 재정의)이 아직 미확정이라 착수 전 별도 논의가 필요하다(`REQ.A.07` 수정 이력 참고). 아래는 목표 설계로 남겨둔다.

- 책임과 경계: `drop_interest_counters WHERE drop_phase = 'OPEN' ORDER BY sell_through_score DESC`. 드롭 상세 원본은 제공하지 않는다(`drop_id`만).
- 상태 변경: 없음.
- 도메인 매핑: Repository → `DropInterestCounterRepository.listByPhaseOrderByScore`.
- `limit`/`cursor` 공용 pagination parameter를 사용한다(2026-07-13 추가).

## API.A.07-06 기다리는 상품 랭킹 조회(구 "오픈 예정 랭킹", 2026-07-14 재정의)

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /v1/rankings/drops/upcoming` |
| operationId | `listUpcomingRanking`(확인 필요: "upcoming"이 phase 필터 제거 후에도 적절한 이름인지 — 외부 계약 변경은 팀 확정 필요, 지금은 유지) |
| 역할 | 드롭 상태 구분 없이, 전체 드롭을 리셋 없는 누적 활성 찜 수(`interest_count`) 내림차순으로 반환한다(`RM.A.07-03`). |
| 유형 | Query |
| 인증 | 불필요 |
| 기본 `limit` | 홈 화면 위젯은 `limit=3`, "전체보기" 페이지는 `limit` 최대 100(2026-07-14 UX 결정) |

- 책임과 경계(2026-07-14 수정): `drop_interest_counters ORDER BY interest_count DESC, drop_id ASC` — `WHERE drop_phase = 'SCHEDULED'` 필터를 제거했다(catalog-service 의존 제거, `REQ.A.07` 수정 이력 참고).
- 도메인 매핑: Repository → `DropInterestCounterRepository.listByInterestCount`(구 `listByPhaseOrderByUpcoming`).
- `limit`/`cursor` 공용 pagination parameter를 사용한다.

## API.A.07-08 실시간 많이 보는 상품 랭킹 조회(신규, 2026-07-14)

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /v1/rankings/drops/trending` |
| operationId | `listTrendingRanking` |
| 역할 | 드롭 상태 구분 없이, 가장 최근에 마감된 KST 3시간 구간(00/03/06/09/12/15/18/21시 경계)의 서로 다른 조회자 수 기준 랭킹을 반환한다(`RM.A.07-06`). |
| 유형 | Query |
| 인증 | 불필요 |
| 기본 `limit` | 홈 화면 위젯은 `limit=3`, "전체보기" 페이지는 `limit` 최대 100 |

- 책임과 경계: `drop_view_rankings WHERE bucket_start = (최신 bucket_start) ORDER BY rank ASC LIMIT ?`. 매 요청마다 `drop_views` 원본을 다시 집계하지 않는다 — 3시간 배치가 미리 계산해둔 스냅샷만 읽는다.
- 상태 변경: 없음.
- 도메인 매핑: Repository → `DropViewRankingRepository.getLatestBucket`.
- `limit`/`cursor` 공용 pagination parameter를 사용한다.
- 확인 필요: 구간 경계 직후(예: 03:00:01)에는 새 구간의 스냅샷이 아직 없을 수 있다 — 이 경우 직전 구간 값을 계속 보여줄지, 빈 리스트를 보여줄지 결정 필요.

## API.A.07-07 드롭 관심도 통계 조회

| 항목 | 값 |
| --- | --- |
| Method / Path | `GET /v1/operator/drops/{dropId}/interest-stats` |
| operationId | `getDropInterestStats` |
| 역할 | DropMong 운영자가 드롭별 찜·조회 통계를 확인한다(`RM.A.07-04`). |
| 유형 | Query |
| 인증 | `Authorization: Bearer <JWT>` + `X-User-Id`/`X-User-Role: operator` (실제 코드 컨벤션은 `contracts/jwt-conventions.md` 참고, role enum은 `CUSTOMER`/`OPERATOR`/`ADMIN`) |
| 노출 범위 | operator (`role=operator`, 2026-07-13 결정으로 브랜드 운영자는 1차 범위 밖) |

- 책임과 경계(2026-07-14 수정): `drop_interest_counters`의 `interest_count`를 드롭 단위로 노출한다. `sell_through_score`(오픈 후 랭킹 후속)는 응답에서 제외했다.
- 확인 필요(2026-07-14 추가): 조회 기능이 다시 범위에 들어오면서, 이 통계 응답에 "최근 3시간 조회자 수" 같은 조회 통계도 같이 넣을지는 아직 정하지 않았다 — 지금은 `interest_count`만 노출한다.
- 1차 스코프 결정(2026-07-13): 브랜드 운영자(`ACTOR.A.07-03`)의 자사 드롭 소유권 검증 메커니즘이 아직 없어 이 Endpoint는 `operator` role로만 제한한다. 브랜드 운영자 접근은 소유권 검증 방식이 정해진 뒤 별도 스코프 확장 또는 별도 API로 다룬다.
- 경로는 `service` 레포 `contracts/jwt-conventions.md`의 "드롭 운영자 API는 `/operator/...` prefix" 규칙을 따랐다. 다만 실제 코드의 유일한 선례인 `backoffice-service`는 같은 role 체크(`HasRole("operator")`)를 쓰면서도 경로는 `/v1/admin/...`을 쓰고 있어 문서-코드 불일치가 있다(확인 필요, 우리는 문서 규칙을 따름).
- 처리 규칙: `X-User-Id`/`X-User-Role` 확인(FastAPI `Header` 의존성, order-service의 `x_user_role: Annotated[UserRole, Header(alias="X-User-Role")]` 패턴과 동일) → role이 `OPERATOR`가 아니면 `403` → `DropInterestCounterRepository.get` 조회.
- 예외: `Authorization`/`X-User-*` 없음·무효 `401`, operator role 아님 `403`, 드롭 없음 `404`.
- 도메인 매핑: Repository → `DropInterestCounterRepository.get`.
- 확인 필요: 브랜드 운영자의 "자사 드롭" 판정 기준을 어느 Context에서 어떻게 조회할지.

## 관측성과 운영(공통)

- 로그: `user_id`(해시), `drop_id`, `operationId`, 처리 결과. 원문 조회수/찜 수는 로그가 아니라 메트릭으로 집계한다.
- Metric: Command 성공/실패율, dedup 적중률(`API.A.07-04`), 낙관적 잠금 충돌률(`API.A.07-01`, `-02`).
- Rate limit: `API.A.07-04`(조회 기록)는 핫키 상황에서 write가 몰릴 수 있어 개별 사용자 기준보다 드롭 기준 rate limit을 우선 검토한다(`POLICY.A.07-03`).

## 확인 필요(공통)

- `/v1/operator/...` vs `/v1/admin/...` 경로 접두사 불일치를 팀과 확정할지(위 참고).
- 커서(`cursor`) 인코딩 방식(어떤 컬럼 기준으로 정렬하고 어떻게 인코딩할지)은 `SD.A.0720`/`SD.A.0730`에서 구현 시 정한다.

## 2026-07-14 수정 이력

- `API.A.07-04`(조회 기록)를 Redis `SETNX` dedup 방식에서 "원문 기록 + 집계 시점 `COUNT(DISTINCT user_id)`" 방식으로 재설계해 후속 스코프에서 1차 구현으로 복귀시켰다.
- `API.A.07-06`(구 "오픈 예정 랭킹")을 "기다리는 상품 랭킹"으로 재정의했다 — `drop_phase=SCHEDULED` 필터를 제거하고 전체 드롭 대상 누적 찜 수 기준으로 바꿨다(catalog-service 의존 제거).
- `API.A.07-08`(실시간 많이 보는 상품 랭킹)을 신설했다. KST 3시간 고정 구간 배치 스냅샷을 읽는 방식이며, `API.A.07-05`(오픈 후 랭킹, order-service 필요)와 달리 카탈로그/오더 서비스 의존이 없다.
- `API.A.07-05`(오픈 후 랭킹)는 후속 스코프로 명시했다 — order-service 연동 방식 미확정.
- `API.A.07-07`(드롭 관심도 통계)의 응답에서 `dropPhase`/`viewCountTotal`/`sellThroughScore`를 제거하고 `interestCount`만 남겼다. 조회 통계를 이 API에 다시 추가할지는 확인 필요로 남겼다.
- 근거: `REQ.A.07`의 "2026-07-14 수정 이력" 1~3차 참고 — 티켓팅/드롭 커머스 유사 사례 조사(오픈전·후 랭킹은 분리하되 신호는 섞지 않음), 사용자 제안(신호 기준 두 랭킹으로 재구성), 부하테스트 대비 논의(고정 3시간 배치로 읽기 경로 단순화) 순으로 반영됐다.

## 2026-07-13 수정 이력

- 인증을 `WebSessionCookie`/`x-required-permissions`(archive의 `A_300_auth` 설계 문서 기준)에서 `X-Principal` 헤더 + `role` claim(`service` 레포 실제 코드 `contracts/jwt-conventions.md`, coupon-service/backoffice-service 기준)으로 정정.
- 경로를 `/api/v1/...`에서 `/v1/...`로 정정(실제 프로젝트는 `/api` 접두사를 쓰지 않음).
- 성공 응답의 `data`/`meta` 제네릭 래핑을 제거. 단일 리소스는 래핑 없이 그대로 반환(coupon-service의 `Policy`/`Coupon` 패턴), 목록은 `{data, pageInfo}`(catalog-service의 `DropListResponse` 패턴), 에러는 `{error: {code, message}, requestId, occurredAt}`로 정정.
- 목록 API(`-03`, `-05`, `-06`)에 `contracts/common/components.yaml`의 공용 `limit`/`cursor` parameter를 추가.

## 2026-07-13 수정 이력(2차, 구현 언어를 Python으로 확정한 뒤)

- 구현 언어를 Python(FastAPI)으로 확정하면서, 위 1차 수정에서 "coupon-service/backoffice-service 기준"으로 정한 `X-Principal` 헤더 인증이 실제로는 **Go 서비스 전용 컨벤션**이었음을 재확인했다. Python 서비스(catalog-service, order-service)는 `X-Principal`을 전혀 쓰지 않고 `Authorization: Bearer <JWT>`(`BearerAuth`) + Gateway가 전달하는 `X-User-Id`/`X-User-Email`/`X-User-Role` 헤더만 쓴다(`contracts/jwt-conventions.md`, `services/order-service/app/main.py`의 `Header(alias="X-User-Id")` 실제 구현으로 확인). 모든 Endpoint의 인증 방식을 이 컨벤션으로 다시 정정했다.
- 1차 수정에서 "단일 리소스는 래핑 없이 반환(coupon-service 기준)"으로 정한 것도 Go 전용 관례였다. Python 공식 계약(`contracts/services/catalog-service/openapi.yaml`의 `DropDetailResponse`, `contracts/services/order-service/openapi.yaml`의 `OrderResponse`)은 단일 리소스도 `{data: ...}`로 감싼다. `InterestResponse`/`ViewRecordResponse`/`DropInterestStatsResponse`를 새로 만들어 `Interest`/`ViewRecordResult`/`DropInterestStats`를 감싸도록 정정했다.
- catalog-service를 직접 확인한 결과 Kafka 연동이 전혀 없는 최소 스텁 상태임을 발견했다. `SD.A.0730`이 전제하는 `catalog.drop.updated` 이벤트 구독은 아직 실제로 검증할 수 없어, 첫 구현 범위를 `Interest` Aggregate(API.A.07-01/-02/-03)로 좁히고 `DropInterestCounter`(랭킹/이벤트) 관련 API(-04, -05, -06, -07)는 다음 단계로 미뤘다.
