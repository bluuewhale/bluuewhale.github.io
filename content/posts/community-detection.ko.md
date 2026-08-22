+++
title = 'Community Detection'
date = '2026-03-23T23:18:53+09:00'
draft = false
math = true
translationKey = 'community-detection'
slug = 'community-detection'
description = 'Modularity가 어떤 발상에서 나온 지표인지, 그리고 이를 최적화하는 Louvain과 Leiden 알고리즘이 어떻게 다른지 살펴봅니다.'
tags = ['Community Detection', 'Graph', 'Network Science']
categories = ['Graph', 'AI']

[cover]
image = 'images/community-detection/image1.png'
hiddenInSingle = true
+++

그래프 안에서 서로 밀접하게 연결된 노드들의 집합, 즉 커뮤니티를 찾아내는 문제를 Community Detection이라고 부릅니다. 일종의 클러스터링 문제로 볼 수 있는데, [GraphRAG](/posts/graphrag/) 같은 방법론이 지식 그래프를 다룰 수 있는 단위로 쪼개는 데 바로 이 기법을 사용합니다. 이 글에서는 Community Detection에서 가장 널리 쓰이는 지표인 Modularity가 어떤 발상에서 나왔는지, 그리고 이를 최적화하는 대표적인 알고리즘인 Louvain과 Leiden이 어떻게 다른지 살펴보겠습니다.

![](/images/community-detection/image1.png)

## Modularity: 커뮤니티가 잘 만들어졌는지 재는 법

커뮤니티를 하나 만들었다고 해봅시다. 이 커뮤니티가 "잘" 만들어졌다는 건 어떤 의미일까요? Community Detection에서 가장 널리 쓰이는 답은 Modularity라는 지표입니다. 직관은 단순합니다. 만약 그래프의 엣지들이 완전히 무작위로 연결되어 있었다면 이 커뮤니티 안에 몇 개의 엣지가 있었을지 계산해두고, 실제로 관찰된 엣지 수가 그 기댓값보다 얼마나 더 많은지를 재는 것입니다. 우연이라고 보기 힘들 정도로 연결이 조밀하다면, 그건 진짜 커뮤니티라고 볼 근거가 됩니다.

두 개의 커뮤니티만 있는 가장 단순한 경우, Modularity는 다음과 같이 정의됩니다.

$$
Q = \frac{1}{2m} \sum_{vw} \left[A_{vw} - \frac{k_v k_w}{2m}\right] \frac{s_v s_w + 1}{2}
$$

기호를 하나씩 풀어보겠습니다. $Q$는 Modularity 값 자체이고, $m$은 그래프 전체의 엣지 개수입니다. $A_{vw}$는 인접 행렬로, $v$번째와 $w$번째 노드가 연결되어 있으면 1, 아니면 0입니다. $k_v$는 $v$번째 노드에 연결된 엣지 개수, 즉 degree입니다.

핵심은 $\frac{k_v k_w}{2m}$입니다. 이 값은 만약 그래프의 모든 연결이 완전히 무작위로 재배치된다면, $v$와 $w$ 사이에 엣지가 존재할 것으로 기대되는 확률, 즉 null model입니다. 연결이 무작위라 해도 degree가 높은 노드일수록 다른 노드와 연결될 가능성이 크다는 점을 반영한 값입니다. 그러니 $A_{vw} - \frac{k_v k_w}{2m}$는 "실제 관찰된 연결"에서 "무작위였다면 기대했을 연결"을 뺀 값, 다시 말해 $v$와 $w$ 사이의 연결이 순전히 우연으로 설명되는 수준을 얼마나 초과하는지를 나타냅니다.

여기에 곱해지는 $\frac{s_v s_w + 1}{2}$는 물리학에서 빌려온 스핀 변수 표기법인데, $v$와 $w$가 같은 커뮤니티에 속하면 1, 아니면 0이 되도록 설계된 항입니다. 즉, 같은 커뮤니티에 속한 노드 쌍에 대해서만 앞서 계산한 "우연을 초과하는 연결 정도"를 모두 더한 것이 Modularity입니다.

이렇게 정의된 $Q$의 부호는 그 자체로 해석이 가능합니다. $Q > 0$이면 무작위로 만들어졌을 때보다 커뮤니티 내부의 연결이 더 조밀하다는 뜻이니 우연이 아니라고 볼 수 있고, $Q = 0$이면 무작위와 다를 바 없는 수준이며, $Q < 0$이면 오히려 무작위보다 연결이 성긴 상태입니다.

## 커뮤니티가 여러 개일 때: 일반화된 형태

방금 본 수식은 커뮤니티가 딱 두 개일 때를 가정한 것입니다. 이를 임의의 개수의 커뮤니티로 확장하면, 스핀 변수 대신 크로네커 델타를 사용한 다음과 같은 형태가 됩니다.

$$
Q = \frac{1}{2m} \sum_{vw} \left[A_{vw} - \frac{k_v k_w}{2m}\right] \delta(c_v, c_w)
$$

![](/images/community-detection/image2.png)

여기서 $\delta(c_v, c_w)$는 $v$와 $w$가 같은 커뮤니티에 속하면 1, 아니면 0을 반환하는 함수로, 앞서 본 스핀 변수 항을 커뮤니티가 몇 개든 상관없이 쓸 수 있도록 일반화한 것뿐입니다. 수식의 나머지 구조와 해석은 두 커뮤니티일 때와 동일합니다.

![](/images/community-detection/image3.png)

## Resolution: "무작위"의 기준을 조절하는 손잡이

여기에 한 가지 파라미터를 더 끼워 넣을 수 있습니다. 바로 Resolution, $\gamma$입니다.

$$
Q = \frac{1}{2m} \sum_{vw} \left[A_{vw} - \gamma \frac{k_v k_w}{2m}\right] \delta(c_v, c_w)
$$

수식을 보면 null model 항인 $\frac{k_v k_w}{2m}$ 앞에 $\gamma$가 곱해져 있는 것을 알 수 있습니다. 이 값을 작게 만들면, "무작위였다면 이 정도는 연결되어 있었을 것"이라고 기대하는 기준 자체가 낮아집니다. 기준이 낮아지면 새로운 노드를 커뮤니티에 추가했을 때 Modularity가 증가하기 위한 조건도 함께 느슨해지고, 결과적으로 상대적으로 약하게 연결된 노드들까지 같은 커뮤니티로 묶일 수 있게 됩니다. $\gamma$를 낮출수록 커뮤니티의 개수는 줄고 크기는 커지는 방향으로 결과가 움직인다고 이해하면 됩니다.

## Louvain 알고리즘

Modularity를 최적화하는 대표적인 알고리즘이 2008년 제안된 Louvain입니다. 크게 두 단계를 반복합니다. 먼저 Local Moving 단계에서는 각 노드를 하나씩 살펴보며, 그 노드를 어느 이웃 커뮤니티로 옮겼을 때 전체 Modularity가 가장 크게 증가하는지를 그리디하게 탐색해 배정합니다. 이렇게 더 이상 개선이 없을 때까지 Local Moving을 반복하고 나면, Aggregation 단계에서 지금까지 찾은 각 커뮤니티를 하나의 노드로 압축해 새로운(더 작은) 그래프를 만듭니다. 이 압축된 그래프에 대해 다시 Local Moving을 수행하는 식으로 두 단계를 계속 반복합니다.

종료 조건은 세 가지 중 하나입니다. Local Moving 단계에서 더 이상 노드가 옮겨지지 않거나, 전체 Modularity가 더는 증가하지 않거나, Aggregation을 거쳐도 그래프 크기가 줄어들지 않는 경우입니다. 결과물로는 계층적인 커뮤니티 구조와 각 노드가 어느 커뮤니티에 속하는지에 대한 할당 정보를 얻습니다.

## Leiden 알고리즘: Louvain의 빈틈을 메우기

Louvain에는 한 가지 허점이 있습니다. 이 알고리즘은 전체 Modularity를 높이는 데에만 집중하다 보니, 같은 커뮤니티로 묶인 노드들끼리 실제로는 서로 연결되어 있지 않은 disconnected community가 만들어질 수 있습니다. Local Moving 과정에서 노드를 이동시키는 순서에 따라 이런 일이 충분히 생길 수 있는데, 커뮤니티라는 이름에 걸맞지 않게 내부적으로 쪼개진 채로 남는 셈입니다.

![](/images/community-detection/image5.png)

Leiden 알고리즘은 바로 이 문제를 겨냥합니다. 커뮤니티 내부의 모든 노드가 서로 도달 가능하다는 것을 보장하도록 설계되었습니다. 방법은 Louvain의 두 단계 사이에 Refinement라는 새로운 단계를 끼워 넣는 것입니다. Local Moving으로 커뮤니티를 찾은 뒤, Refinement 단계에서는 각 커뮤니티 내부의 모든 노드를 일단 각자 독립된 커뮤니티로 되돌립니다. 그런 다음 그 안에서 실제로 서로 잘 연결되어 있는 노드들끼리만 다시 하나로 묶고, 연결이 없는 노드는 그대로 혼자 남겨둡니다. 이렇게 정제된 결과를 놓고서야 Aggregation으로 넘어갑니다. Local Moving, Refinement, Aggregation 세 단계를 반복하는 구조 덕분에, Leiden은 Louvain과 비슷한 계산 비용으로도 내부적으로 실제 연결된 커뮤니티만을 결과로 내놓습니다.

## References
- [Modularity (networks)](https://en.wikipedia.org/wiki/Modularity_(networks))
- [네트워크 데이터 분석 - Communities](https://sanghn.tistory.com/21)
- [네트워크 데이터 분석 - Community detection: Modularity](https://sanghn.tistory.com/27)
