# DropMong 테스트와 릴리즈 계획

작성일: 2026-07-02

이 문서는 구현 후 어떤 테스트와 릴리즈 검증으로 설계를 완료 처리할지 정의한다.

## 1. 테스트 계층

| 계층 | 저장소 | 목적 |
| --- | --- | --- |
| unit | `services` | domain state transition, idempotency, event handler |
| contract | `services` | OpenAPI request/response와 error envelope |
| integration | `services` | DB transaction, outbox relay, consumer idempotency |
| synthetic | `e-gitops` | 배포된 smoke와 journey |
| load | `e-gitops` 또는 k6 script | drop-open traffic과 overload |
| release | `e-gitops` | canary와 rollback |
| infra | `infra`, `e-gitops` | ingress, storage, ECR, cluster wiring |

## 2. 필수 테스트 시나리오

| ID | 시나리오 | 성공 기준 |
| --- | --- | --- |
| T01 | catalog 공개 조회 | 응답 OK, cache header/tag 존재 |
| T02 | order 생성 | `PENDING_PAYMENT`, reservation active |
| T03 | oversell zero | confirmed order가 quantity를 넘지 않음 |
| T04 | order idempotency replay | 같은 key와 payload가 같은 response 반환 |
| T05 | idempotency conflict | 같은 key와 다른 payload가 409 반환 |
| T06 | payment approved | order가 `CONFIRMED`가 됨 |
| T07 | payment failed | reservation release, order cancel |
| T08 | payment delayed beyond TTL | order expired, late approval이 confirm하지 않음 |
| T09 | outbox relay restart | relay restart 후 pending event publish |
| T10 | duplicate event delivery | consumer result가 idempotent하게 유지 |
| T11 | notification outage | checkout 성공, lag 관측 |
| T12 | Kafka lag scale | KEDA가 consumer를 scale하고 lag를 해소 |
| T13 | canary rollback | SLO 위반 시 rollback |
| T14 | security ownership | customer가 다른 order를 읽을 수 없음 |

## 3. k6 부하 프로필

### 드롭 오픈 스파이크

fixed VU만 사용하지 말고 open-model arrival rate를 사용한다.

```text
warm catalog -> wait until open -> ramp arrival rate -> hold -> cool down
```

메트릭:

- admitted RPS
- queued RPS
- rejected RPS
- accepted order p95/p99
- sold-out response rate
- oversell count after test

### 재시도 폭주

동작:

- 같은 `Idempotency-Key`를 여러 번 전송
- client timeout과 retry를 섞음
- jitter group과 no-jitter group 포함

기대 결과:

- duplicate order 없음
- idempotency replay metric 증가
- conflict metric은 의도적으로 다른 payload에서만 증가

### 결제 실패와 지연

동작:

- approve group
- fail group
- TTL을 넘기는 delay group

기대 결과:

- approved order는 confirm
- failed order는 cancel
- delayed order는 expire되고 late approval은 안전하게 처리

### 알림 장애

동작:

- notification consumer를 중지하거나 0으로 scale
- checkout journey 실행
- consumer 복구

기대 결과:

- checkout은 정상 유지
- consumer 중단 중 lag 증가
- 복구 후 notification catch up

## 4. 릴리즈 게이트

service 변경 merge 전:

- unit test 통과
- contract test 통과
- DB migration 하위 호환
- idempotency test 통과
- event handler duplicate test 통과

외부 route 활성화 전:

- health와 readiness 통과
- internal smoke 통과
- metric scrape 확인
- log가 Loki에서 보임
- trace가 Tempo에서 보임

100 percent rollout 전:

- canary analysis 통과
- oversell 없음
- error rate가 threshold 이하
- accepted latency가 threshold 이하
- outbox와 Kafka lag 안정

## 5. Canary 계획

`order-service`와 `payment-service` 대상:

```text
setWeight 5
pause analysis
setWeight 20
pause analysis
setWeight 50
pause analysis
setWeight 100
```

Prometheus 분석:

- canary error rate
- accepted p95/p99
- oversell count
- outbox pending age
- Kafka lag
- DLQ depth

## 6. Rollback 계획

즉시 rollback할 조건:

- `oversell_count > 0`
- error rate가 SLO 위반
- order create p95/p99가 threshold 위반
- canary payment handler failure 증가
- canary version에서만 DLQ 증가

Rollback 절차:

1. analysis 실패 시 Argo Rollouts가 canary를 자동 abort하게 둔다.
2. 자동화가 실패하면 stable ReplicaSet을 수동 promote한다.
3. 추가 rollout을 동결한다.
4. log, trace, metric, event id를 보존한다.
5. `workspaces/docs/evidence/dropmong` 아래에 incident evidence를 생성한다.
6. root cause와 test gap을 담은 follow-up issue를 연다.

## 7. 릴리즈 준비 완료 정의

- [ ] 제품 범위와 서비스 경계 승인
- [ ] ADR 승인
- [ ] API 계약이 `services/contracts`에 생성 또는 작성됨
- [ ] Event 계약이 outbox와 함께 구현됨
- [ ] DB migration review 완료
- [ ] Synthetic smoke 통과
- [ ] Drop-open load test가 oversell 0으로 통과
- [ ] Canary rollback 테스트 완료
- [ ] Runbook review 완료
- [ ] Dashboard와 alert 존재
