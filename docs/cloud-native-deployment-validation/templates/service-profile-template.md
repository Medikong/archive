---
title: 서비스 특성 기록 템플릿
type: service-profile-template
status: template
service_name: TBD
owner: TBD
namespace: TBD
primary_workloads:
  - TBD
current_deployment_strategy: TBD
candidate_deployment_strategies:
  - Rolling
  - Canary
  - Blue-Green
  - Shadow
  - Feature Flag
  - A/B
created: TBD
updated: TBD
---

# 서비스 특성 기록 템플릿

## 서비스 특성

| 분류 | 질문 | 답변 | 배포 전략 영향 |
| --- | --- | --- | --- |
| 상태 보유 | 서버 내부 상태, 세션, local cache에 의존하는가? | TBD | TBD |
| DB 보유 | 서비스 소유 DB 또는 migration이 있는가? | TBD | TBD |
| 정합성 | 중복 처리나 순서 역전이 치명적인가? | TBD | TBD |
| 외부 연동 | 결제, 알림, 배송, 인증 provider와 연결되는가? | TBD | TBD |
| 하위호환성 | 구버전/신버전 API와 event가 동시에 동작 가능한가? | TBD | TBD |
| 롤백 가능성 | image, route, flag, DB를 되돌릴 수 있는가? | TBD | TBD |
| 트래픽 규모 | canary 판단에 충분한 요청 수가 있는가? | TBD | TBD |
| 관측 가능성 | 성공/실패를 metric, log, trace로 판단 가능한가? | TBD | TBD |

## 위험 시나리오

| 위험 | 발생 조건 | 사용자 영향 | 완화 장치 |
| --- | --- | --- | --- |
| TBD | TBD | TBD | TBD |

## 1차 전략 가설

- 우선 후보: TBD
- 보조 후보: TBD
- 제외 후보: TBD
- 판단 근거: TBD
