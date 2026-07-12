# High-Concurrency Practices of Redis: Snap-Up System

- 회사: Alibaba Cloud
- 원문: https://www.alibabacloud.com/blog/high-concurrency-practices-of-redis-snap-up-system_597858
- 접근일: 2026-07-09
- 주제: 플래시 세일(스냅업) 시스템의 계층형 트래픽 필터링, Lua 원자적 재고 차감, TairString 경계 카운터, 분산 락
- 확인 방식: WebFetch로 원문 요약 확인 (Alibaba Cloud Community 공식 블로그)

## 원문 확인 내용

- 플래시 세일은 "프로모션 전(상세페이지 새로고침 폭증) → 프로모션 중(주문 요청 피크) → 프로모션 후(주문 조회/취소)" 3단계로 트래픽 패턴이 다르다.
- 요청을 클라이언트 차단 → 서버 로컬 캐시 → Redis 계층 → DB 순으로 단계적으로 걸러내, 최종적으로 DB에 도달하는 요청을 최소화하는 계층형 아키텍처를 쓴다. 문서 예시에서는 읽기/쓰기 분리 인스턴스가 60만 QPS까지 무효 주문을 걸러내고, 마스터-레플리카 인스턴스가 Lua 스크립트로 10만 QPS의 재고 차감을 처리한다.
- 재고 차감은 Lua 스크립트로 "재고 확인 → 감소"를 하나의 원자적 연산으로 묶어 음수(초과 판매)를 방지한다. 다만 Lua 스크립트는 파싱/VM 처리 비용 때문에 일반 명령어보다 느리므로, 간단한 조건문 수준으로만 쓰기를 권장한다.
- Alibaba Cloud의 TairString(Redis 호환 확장 자료구조)은 `exIncrBy`로 상한/하한 경계를 가진 카운터를 제공해, 경계를 넘는 증감 요청을 자체적으로 거부한다. `exCAS`(Compare And Set)로 버전 기반 낙관적 락도 지원한다.
- 분산 락은 `SET NX EX`로 구현하고, 락 해제 시 CAD(Compare And Delete)로 락을 건 주체만 해제 가능하게 해 다른 프로세스가 실수로 락을 해제하는 문제를 방지한다.

## 적용 아이디어

- 드롭 오픈 직후 특정 인기 드롭에 조회/찜 요청이 몰리는 상황에서, 이 글의 "클라이언트 → 로컬 캐시 → Redis → DB" 계층형 필터링을 interest-service의 조회 API에 적용할 수 있다. 예: 오픈 직후 N초간은 조회수 증가 요청을 애플리케이션 서버 로컬 캐시에서 먼저 합산한 뒤 배치로 Redis에 반영(Toss 사례와 같은 방향, `companies/domestic/toss/README.md` 참고).
- `SET NX EX`가 "중복 방지"(`analysis/01-view-count-dedup.md`)와 "동시성 제어"(이 문서) 양쪽에 쓰인다는 점은, interest-service가 이미 dedup 목적으로 채택한 Redis 패턴을 카운터 동시성 문제에도 일관되게 확장할 수 있다는 근거가 된다.

## 확인 필요

- TairString은 Alibaba Cloud 전용 확장으로, 팀이 쓰는 오픈소스 Redis에는 없는 기능이다. 동일한 경계 카운터가 필요하다면 `INCR` 후 애플리케이션 레벨에서 상한 체크 + 초과 시 `DECR`로 되돌리는 방식이나 Lua 스크립트로 직접 구현해야 한다.
