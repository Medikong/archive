# ADR-001 구매 서비스 언어 선택

작성일: 2026-07-14
상태: 제안

## 배경

현재 catalog, order, payment, notification 서비스는 Python FastAPI로 구매 내부 회귀를 통과했다. order-service는 동시 주문, PostgreSQL transaction, Kafka 처리와 spike latency 때문에 Go 전환 후보가 될 수 있다.

## 제안

- 전체 서비스를 한 번에 Go로 재작성하지 않는다.
- 실제 부하 증거가 있을 때 order-service를 첫 후보로 검토한다.
- catalog, payment mock, notification은 현재 언어로 요구사항을 충족하면 유지한다.
- 언어를 바꿔도 REST path, OpenAPI, Kafka event, metric과 DB 불변조건은 유지한다.

## Go 전환 시작 조건

- 재현 가능한 spike 테스트에서 Python runtime 또는 worker 모델이 병목으로 확인된다.
- DB pool, SQL, lock, Kafka lag 문제를 먼저 분리했다.
- 기존 REST·Kafka·DB 불변조건을 새 구현에도 적용할 수 있다.
- rollback 가능한 병행 배포 또는 트래픽 전환 계획이 있다.

## 검증

- 기존 정상 구매·결제 실패·동시성 gate 모두 통과
- oversell 0, REST와 consumer 멱등성 유지
- 같은 환경과 fixture에서 p95/p99와 자원 사용 비교
- trace, log correlation, metric과 Kafka lag 유지

## 재검토 조건

실제 부하 결과, 서비스 책임, 재고 저장 구조가 바뀌면 다시 검토한다.
