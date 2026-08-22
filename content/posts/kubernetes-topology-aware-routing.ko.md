+++
title = 'Kubernetes Topology Aware Routing'
date = '2025-11-14T10:14:05+09:00'
draft = false
translationKey = 'kubernetes-topology-aware-routing'
slug = 'kubernetes-topology-aware-routing'
description = 'kube-proxy가 EndpointSlice의 zone hint를 참조해 같은 가용 영역의 Pod를 우선 호출하도록 만드는 Kubernetes Topology Aware Routing의 동작 원리와 활성화 조건을 정리합니다.'
tags = ['Kubernetes', 'Networking', 'AWS', 'EKS']
categories = ['Kubernetes', 'Networking']

[cover]
image = 'images/kubernetes-topology-aware-routing/image1.png'
hiddenInSingle = true
+++

EKS의 기본 서비스 라우팅은 클러스터 안의 Pod들에게 트래픽을 랜덤하게, 혹은 라운드로빈으로 분배합니다. 그러다 보니 요청이 다른 가용 영역(AZ)의 Pod로 자주 날아가게 되는데, AWS VPC에서는 AZ 간 데이터 전송에 과금이 붙기 때문에 이런 cross-AZ 트래픽은 비용 증가와 지연 시간 증가로 곧장 이어집니다. Kubernetes 1.24에서 도입된 Topology Aware Routing(TAR)은 바로 이 문제를 겨냥한 기능입니다. Pod 간 통신이 발생할 때, 가능하다면 같은 AZ에 있는 Pod를 우선 호출하도록 네트워크 정책을 조정합니다.

![](/images/kubernetes-topology-aware-routing/image1.png)

## EKS에서의 동작 방식

동작 자체는 단순합니다. EndpointSlice Controller가 엔드포인트를 추가할 때 Pod의 zone 정보를 바탕으로 `hints`를 함께 기록해두고, kube-proxy는 Service 트래픽을 처리할 때 이 `hints`를 읽어 같은 zone에 있는 엔드포인트를 우선 선택합니다. 같은 zone에 쓸 만한 엔드포인트가 부족하면, 기본 동작으로 cross-AZ fallback이 일어납니다.

## 조금 더 자세히 들여다보면

### 1. 노드에 zone 정보 라벨링

Kubernetes는 리소스가 실행되는 머신의 리전과 존 정보를 메타데이터에 담습니다. `topology.kubernetes.io/region`과 `topology.kubernetes.io/zone` 라벨이 여기에 해당하는데, 클라우드 서비스 프로바이더가 제공하는 Kubernetes 환경이라면 이 라벨링이 자동으로 이뤄지는 경우가 많습니다.

### 2. EndpointSlice와 hints

TAR의 실질적인 동작은 EndpointSlice 오브젝트의 `hints` 필드에서 시작됩니다. [EndpointSlice Controller는 엔드포인트를 생성할 때, Pod가 위치한 zone과 매칭되는 정보를 hint로 함께 적어 넣습니다](https://github.com/kubernetes/endpointslice/blob/release-1.28/reconciler.go#L265).

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-abc
  labels:
    kubernetes.io/service-name: my-service
addressType: IPv4
endpoints:
- addresses:
  - 10.0.1.5
  conditions:
    ready: true
  zone: ap-northeast-2a
  hints:
    forZones:
    - name: ap-northeast-2a
- addresses:
  - 10.0.2.8
  conditions:
    ready: true
  zone: ap-northeast-2b
  hints:
    forZones:
    - name: ap-northeast-2b
```

kube-proxy는 서비스 라우팅 시점에 이 hint를 참조해서 같은 zone의 엔드포인트를 우선 선택합니다.

여기서 몇 가지 용어를 짚고 넘어가면 이해가 쉽습니다. EndpointSlice Controller는 EndpointSlice의 생성과 갱신을 책임지는 컨트롤러로, Service나 Pod 같은 리소스의 변화를 추적해 EndpointSlice를 업데이트합니다. EndpointSlice 자체는 Service가 바라보는 Pod IP 목록을 담는 리소스라고 이해하면 됩니다. 하나 이상의 엔드포인트를 포함하고, 각 엔드포인트는 Pod IP와 메타데이터, 그리고 앞서 본 `hints` 필드를 갖습니다. kube-proxy는 이 EndpointSlice를 구독하면서 `hints.forZones`를 확인해 라우팅 규칙을 만듭니다. kube-proxy는 Kubernetes 워커 노드에서 동작하는 데몬으로, Service의 가상 IP와 로드밸런싱을 실제 네트워크 경로로 구현하는 역할을 합니다. 내부적으로는 iptables 같은 커널의 네트워크 기능을 활용하고, Topology Aware Hint가 있으면 같은 zone의 엔드포인트를 우선하도록 라우팅 규칙을 구성합니다.

### 3. Fallback

같은 zone에 건강한(healthy) 엔드포인트가 충분하지 않으면, kube-proxy는 zone을 넘어선 엔드포인트로 트래픽을 보냅니다.

## CNI와의 관계

TAR는 CNI와도 밀접하게 얽혀 있습니다. kube-proxy는 Service에서 Pod로의 트래픽 분산을 담당하며, EndpointSlice의 TAR hint를 보고 인접한 zone의 엔드포인트를 우선하는 규칙을 만듭니다. 반면 CNI(Container Network Interface)는 Pod의 네트워크 인터페이스, IP 할당, 라우팅 자체를 담당하는 플러그인으로, Calico나 Cilium, Flannel, Weave Net 같은 것들이 여기 해당합니다.

대부분의 CNI(Calico, Flannel 등)는 kube-proxy를 그대로 사용하기 때문에 TAR 기능도 별다른 문제 없이 함께 씁니다. 문제는 Cilium처럼 kube-proxy replacement 같은 자체 모드를 제공하는 CNI입니다. 이런 모드를 쓰면 kube-proxy 자체가 개입하지 않고 자체 알고리즘으로 라우팅이 이뤄지기 때문에 EndpointSlice의 hint가 의미를 잃고, 결과적으로 TAR도 적용되지 않습니다. 참고로 EKS의 기본 CNI인 Amazon VPC CNI는 kube-proxy 기반이라 TAR를 문제없이 지원합니다.

## 활성화 방법

Kubernetes 1.24 이상의 EKS 클러스터가 필요하고, Service 리소스에 주석을 하나 추가하면 됩니다. 버전 1.24~1.26에서는 `service.kubernetes.io/topology-aware-hints: "auto"`를, 1.27 이상에서는 `service.kubernetes.io/topology-mode: "Auto"`를 사용합니다.

## TAR가 적용되지 않을 수 있는 경우들

몇 가지 조건에서는 TAR가 아예 적용되지 않거나 기대만큼 동작하지 않을 수 있습니다.

zone 개수보다 엔드포인트 개수가 적으면 애초에 TAR가 적용되지 않습니다. zone별로 엔드포인트 분포가 지나치게 불균형해도 마찬가지입니다. 노드 중 단 하나라도 `topology.kubernetes.io/zone` 라벨이 없거나 할당 가능한 CPU 지표를 제공하지 않으면 TAR는 동작하지 않고, EndpointSlice 안의 엔드포인트 중 하나라도 `hints.forZones`가 비어 있어도 마찬가지입니다.

Horizontal Pod Autoscaler(HPA)를 쓰는 환경도 주의가 필요합니다. HPA가 Pod를 추가할 때는 Topology Spread Constraints 덕분에 여러 zone에 고르게 분산되지만, Pod를 줄일 때 개입하는 Deployment Controller는 zone 간 균형을 신경 쓰지 않고 무작위로 종료합니다. 이 과정이 반복되면 zone 간 불균형이 누적되어 결국 TAR가 꺼질 수 있습니다. [Descheduler](https://github.com/kubernetes-sigs/descheduler)를 활용하면 이 불균형 문제를 어느 정도 우회할 수 있습니다. 애초에 부하를 고르게 분산하려면 TAR와 함께 [Pod Topology Spread Constraint](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)를 같이 설정하는 것이 좋습니다.

## 적용 조건: Pod가 AZ에 고르게 분포해야 한다

가장 핵심적인 조건은 Pod가 모든 AZ에 고르게 분포해야 한다는 점입니다. 각 AZ가 가져야 할 최소 Pod 개수는 다음 식으로 계산됩니다.

```
전체 Pod 개수 * (해당 zone의 CPU 비율) * (1/1.2)
```

이 조건을 만족하지 못하면 TAR는 활성화되지 않습니다. 여기서 등장하는 1.2라는 상수는 [Kubernetes 소스 코드에 고정된 값](https://github.com/kubernetes/kubernetes/blob/v1.24.12/pkg/controller/endpointslice/topologycache/topologycache.go#L249)이라 사용자가 바꿀 수 없습니다.

## References
- [Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Cost Optimization - Networking - Amazon EKS](https://docs.aws.amazon.com/eks/latest/best-practices/cost-opt-networking.html)
