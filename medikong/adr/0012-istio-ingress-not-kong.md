---
id: ADR-0012
title: "DropMong은 Kong 대신 Istio Ingress Gateway를 사용한다"
status: proposed
date: 2026-07-02
areas:
  - infra
  - gitops
  - traffic
repos:
  - workspaces
  - e-gitops
  - infra
decision_drivers:
  - 단순한 traffic 설명
  - canary 통합
  - mTLS와 telemetry
related:
  - ../08-infra-deployment.md
links: []
supersedes: []
superseded_by: null
---

# ADR 0012: DropMong은 Kong 대신 Istio Ingress Gateway를 사용한다

## 상태

Proposed

## 배경

기존 문서와 manifests에는 Kong과 Istio가 함께 등장한다. DropMong 전환에서는 외부 진입, traffic split, telemetry, mTLS, canary를 하나의 이야기로 설명해야 한다.

## 결정

DropMong target architecture는 AWS NLB에서 Istio Ingress Gateway로 진입한다. Kong은 target path에서 제거하거나 archive한다.

## 대안

| 대안 | 장점 | 단점 | 판단 |
| --- | --- | --- | --- |
| Kong 유지, Istio 내부 mesh만 사용 | 기존 작업 재사용 | 외부 route와 mesh route가 이중으로 설명된다 | 기각 |
| Kong만 사용 | edge gateway가 단순 | mTLS, canary, telemetry 설계가 별도로 흩어진다 | 기각 |
| Istio Gateway 사용 | route, mTLS, canary가 한 경로에 모인다 | 기존 Kong manifest 제거 필요 | 채택 |

## 결과

좋아지는 점:

- traffic routing과 rollout 분석을 같은 경로에서 설명할 수 있다.
- Argo Rollouts와 Istio traffic split 연결이 자연스럽다.
- mTLS strict rollout 문서와도 맞다.

비용:

- `e-gitops` Kong annotations와 routes를 정리해야 한다.
- `infra`의 NLB target과 service exposure를 확인해야 한다.

## 후속 작업

| 상태 | 작업 | 담당 | 연결 문서 |
| --- | --- | --- | --- |
| 계획됨 | Kong route 사용처 inventory | 미지정 | ../08-infra-deployment.md |
| 계획됨 | Istio Gateway와 VirtualService target manifest 작성 | 미지정 | ../08-infra-deployment.md |
