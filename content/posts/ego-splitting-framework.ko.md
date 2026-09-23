+++
title = 'Ego-Splitting Framework: from Non-Overlapping to Overlapping Clusters'
date = '2026-03-27T16:45:22+09:00'
draft = false
math = true
translationKey = 'ego-splitting-framework'
slug = 'ego-splitting-framework'
description = '노드의 자아(ego)를 소속 커뮤니티 수만큼 쪼개는 persona graph 기법으로, 평범한 비중첩 클러스터링 알고리즘만으로 중첩 클러스터링을 풀어내는 Ego-Splitting Framework를 정리합니다.'
tags = ['Community Detection', 'Graph', 'Clustering']
categories = ['AI', 'Graph']

[cover]
image = 'images/ego-splitting-framework/image2.png'
hiddenInSingle = true
+++

> 이 글은 구글이 2017년 KDD에서 발표한 논문 *Ego-Splitting Framework: from Non-Overlapping to Overlapping Clusters*를 정리한 글입니다.

## 왜 비중첩 클러스터링만으로는 부족한가

현실의 네트워크에는 보통 중간 크기(대략 100명 안팎)의 커뮤니티가 다수 존재하고, 하나의 노드가 여러 커뮤니티에 동시에 걸쳐 있는 경우가 흔합니다. 그런데 기존의 비중첩(non-overlapping) 클러스터링 알고리즘들은 각 노드를 정확히 하나의 커뮤니티에만 배정하기 때문에, 이러한 실제 네트워크 구조를 제대로 반영하지 못합니다.

물론 중첩(overlapping) 클러스터링을 시도하는 알고리즘들도 이미 여럿 있었지만, 대체로 지나치게 복잡하거나 유연성이 떨어졌고, 이론적인 보장도 부족했습니다.

그래프 전체를 살펴보는 macroscopic 수준에서도 문제가 있습니다. 핵심적인 허브 노드 하나 때문에 서로 이질적인 집단들이 통째로 하나의 거대 커뮤니티로 묶여버리는 현상이 자주 일어납니다. SNS의 팔로우 관계 그래프에 커뮤니티 탐지를 실행한다고 생각해보면 쉽게 이해할 수 있습니다. 일론 머스크는 기업인이자 정치인이자 과학자입니다. 그를 중심으로 놓고 클러스터링을 하면, 원래는 서로 거의 겹치지 않는 기업인 집단과 정치인 집단, 과학자 집단이 그와 연결되어 있다는 이유만으로 같은 커뮤니티로 묶일 수 있습니다.

## 사전 지식: Ego-net

논문을 이해하는 데 필요한 핵심 개념이 ego-net입니다. 노드 $u$의 ego-net($G[N_u]$)은 $u$와 직접 연결된(1-hop) 이웃 노드들($N_u$)만으로 구성된 유도된 부분그래프(induced subgraph)입니다. 여기서 $u$ 자신은 포함되지 않습니다.

![](/images/ego-splitting-framework/image1.png)

여기서 유도된 부분그래프가 무엇인지 잠시 살펴보겠습니다. 그래프 $G$ 안의 노드 부분집합 $S$에 대한 유도된 부분그래프 $G[S]$는 두 가지 성질을 만족합니다. 첫째, 노드 구성이 $S$와 완전히 동일합니다. 둘째, $S$ 내부의 연결만 유효하고 $S$ 밖으로 나가는 연결은 제외됩니다.

논문에서 다루는 비중첩 클러스터링 알고리즘 $A$는 그래프 $G$를 입력받아 $A(G) = (V_1, ..., V_t)$ 형태로 노드 집합을 분할합니다. 이때 서로 다른 두 클러스터 $V_i$와 $V_j$는 겹치지 않아야 하고($V_i \cap V_j = \emptyset$), 모든 노드는 반드시 어느 한 클러스터에는 속해야 합니다($V_1 \cup ... \cup V_t = V$). 저자들은 이 과정에 어떤 비중첩 클러스터링 알고리즘을 사용해도 무방하다고 설명합니다. 다만, 논문에서는 Absolute Potts Model 기반의 label propagation 알고리즘을 사용했습니다. 확장성이 매우 높아 대규모 그래프의 분산 처리에 적합하고, ego-net을 다루는 선행 연구들에서도 흔히 쓰이던 방법론이기 때문입니다.

## 핵심 아이디어: 자아를 쪼개기

논문에서 제안하는 핵심 아이디어는 다음과 같습니다. microscopic 수준, 즉 노드 하나하나의 단위에서 비중첩 클러스터링 알고리즘을 적용해, 그 결과로 중첩 클러스터링을 구현하자는 것입니다.

![](/images/ego-splitting-framework/image2.png)

다시 일론 머스크의 예시로 돌아가 보면, 그의 자아(ego)를 소속된 커뮤니티 수만큼 쪼개서(persona) 각 집단에 하나씩 포함시키는 것이 아이디어의 핵심입니다. 기업인 집단(트위터의 다른 CEO들 등)을 만들 때는 머스크의 persona 하나(a1)를 연결하고, 정치인 집단을 만들 때는 또 다른 persona(a2)를 연결하는 식입니다. 알고리즘은 크게 두 단계, Local Ego-Net Analysis와 Global Graph Partitioning으로 구성됩니다.

### Local Ego-Net Analysis

먼저 모든 노드 $u$에 대해 ego-net $G[N_u]$를 계산합니다. 그런 다음 이 ego-net에 비중첩 클러스터링 알고리즘 $A^l$을 적용해, $u$의 이웃들을 여러 파티션으로 나눕니다.

$$A^l(G[N_u]) = \{N^1_u, N^2_u, ..., N^t_u\}, \quad t_u = np(A^l, G[N_u])$$

이렇게 나눈 각 파티션에 대해, 원래 노드 $u$의 복제본, 즉 persona를 하나씩 만듭니다.

![](/images/ego-splitting-framework/image3.png)

persona를 모두 만든 뒤에는 원본 노드 $u$ 자체를 그래프에서 제거합니다. 이 과정을 그래프의 모든 노드에 대해 반복하고 나면, persona graph라는 새로운 그래프가 완성됩니다.

### Global Graph Partitioning

이제 이 persona graph에 다시 한 번 비중첩 클러스터링 알고리즘 $A^g$를 적용합니다. 여기서 얻은 클러스터링 결과의 persona 노드들을 원래의 노드로 다시 매핑해주면, 하나의 원본 노드가 여러 클러스터에 속하는 중첩 클러스터링 결과를 얻을 수 있습니다.

![](/images/ego-splitting-framework/image4.png)

전체 과정을 요약하면, 로컬 수준에서 노드의 자아를 소속 커뮤니티 수만큼 쪼개서 그래프를 확장한 뒤, 그 확장된 그래프에 익숙한 비중첩 클러스터링을 한 번 더 실행하고, 마지막에 다시 원래 노드로 되돌리는 것입니다. 즉, 비중첩 클러스터링 알고리즘 자체를 수정하지 않고도, 로컬 분석과 전체 그래프 분할에 각각 적용하여 중첩 클러스터링 문제를 해결할 수 있습니다.

## 왜 이 방식이 의미 있는가

가장 큰 장점은 복잡한 중첩 클러스터링 문제를, 검증된 단순한 비중첩 클러스터링 알고리즘으로 환원해서 풀어낸다는 점입니다. 또한 비중첩 클러스터링 알고리즘을 자유롭게 선택할 수 있다는 장점이 있습니다.

구조 자체도 MapReduce 같은 대규모 분산 처리 환경에 잘 맞습니다. 그래프 크기가 100배 커져도 실행 시간은 10배 정도밖에 늘어나지 않을 만큼 확장성이 뛰어납니다.

![](/images/ego-splitting-framework/image5.png)

다음으로 성능 비교 결과를 살펴보겠습니다. DEMON처럼 ego-net 기반의 대표적인 기존 중첩 클러스터링 알고리즘과 비교했을 때, 벤치마크에서 훨씬 우수한 성능을 보였습니다.

![](/images/ego-splitting-framework/image6.png)

뿐만 아니라 실제 그래프 데이터셋을 대상으로 한 실험에서도 가장 우수한 성능을 기록했습니다.

![](/images/ego-splitting-framework/image7.png)

## 마무리

Ego-Splitting Framework의 흥미로운 점은 기존 비중첩 클러스터링 알고리즘을 활용하여 중첩 클러스터링 문제를 해결한다는 것입니다. 노드를 커뮤니티 수만큼의 persona로 나누고, 로컬 분석과 전체 그래프 분할에 비중첩 클러스터링을 각각 적용합니다. 이를 통해, 기존 알고리즘을 수정하지 않고도 더 복잡한 중첩 클러스터링 문제를 다룰 수 있습니다.
