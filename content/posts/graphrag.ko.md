+++
title = 'GraphRAG'
date = '2026-03-13T10:26:08+09:00'
draft = false
translationKey = 'graphrag'
slug = 'graphrag'
description = '지식 그래프와 커뮤니티 탐지를 결합해, 코퍼스 전체를 아울러야 답할 수 있는 질문에 강한 GraphRAG의 인덱싱·쿼리 파이프라인과 평가 결과를 정리합니다.'
tags = ['RAG', 'AI', 'Knowledge Graph', 'LLM']
categories = ['AI', 'RAG']

[cover]
image = 'images/graphrag/image6.png'
hiddenInSingle = true
+++

> 이 글은 마이크로소프트 리서치의 논문 [From Local to Global: A GraphRAG Approach to Query-Focused Summarization](https://arxiv.org/pdf/2404.16130)을 정리한 글입니다.

RAG는 신뢰할 수 있는 문서 집합을 구축해두고, 질문이 들어오면 관련 문서를 찾아 LLM에게 근거로 제공해 답을 생성하게 하는 방식입니다. 인덱싱(문서를 chunking해서 검색 가능한 형태로 저장), Retrieval(질문과 관련된 문서를 탐색), Generator(탐색된 문서와 질문을 바탕으로 답변 생성)가 기본 구성 요소입니다.

![](/images/graphrag/image1.png)

## 기존 RAG의 한계

RAG의 발전은 대체로 "질문의 의도를 어떻게 더 정확히 파악할 것인가"와 "질문과 연관된 문서를 어떻게 더 잘 찾을 것인가"에 초점을 맞춰 이뤄져 왔습니다. 문서 하나하나를 더 잘 찾는 데는 강했지만, 정작 문서들 사이의 관계나 연결성에는 상대적으로 관심이 적었습니다.

![](/images/graphrag/image2.png)

이 공백이 특히 두드러지는 지점이 Query-Focused Summarization입니다. "이 데이터셋의 전반적인 주제가 뭔가요?"나 "최근 10년간 연구의 핵심 트렌드는 무엇인가요?" 같은 질문은 코퍼스 전체를 조망해야만 답할 수 있는데, 관련 문서 몇 개만 찾아서 보여주는 기존 RAG 방식으로는 애초에 다루기 어려운 유형입니다. GraphRAG는 바로 이 지점을 겨냥합니다.

## 배경 지식: 지식 그래프

지식 그래프는 지식 베이스, 즉 서로 다른 두 객체의 종류와 그 사이의 관계를 담은 triplet들을 그래프 형태로 표현한 것입니다.

![](/images/graphrag/image3.png)

물리적으로 적은 양의 텍스트로도 많은 정보를 압축해서 담을 수 있고, entity 간의 연결을 직접 확인할 수 있으며, 필요에 따라 노드와 관계의 종류를 조절할 수 있다는 장점이 있습니다.

지식 그래프를 활용해 질문에 답하는 과업으로는 Knowledge Base Question Answering(KBQA)이 먼저 자리 잡고 있었습니다. 텍스트로 된 질문에 텍스트로 답을 낸다는 점에서 GraphRAG와 비슷해 보이지만, 둘은 성격이 꽤 다릅니다. KBQA는 Freebase나 Wikidata 같은 구조화된 지식 베이스에서 정확한 entity를 찾아내는 것이 목표이고, 질문을 SPARQL 같은 논리 형식으로 변환해 규칙 기반으로 추론합니다. 반면 GraphRAG는 문서 컬렉션을 그래프로 변환한 뒤, 그래프 탐색과 LLM의 생성을 결합해 자연어 텍스트로 답을 만들어냅니다. 이런 의미에서 GraphRAG는 KBQA를 일반화한 형태로 볼 수 있습니다.

지식 그래프를 만드는 실무적인 방법은 LLM에 문서를 입력하고, 엔티티와 관계로 구성된 triplet을 뽑아내라고 프롬프팅하는 것입니다. 이렇게 생성된 triplet들을 모아 전체 그래프를 구축합니다.

## 배경 지식: 커뮤니티 탐지

지식 그래프를 만들고 나면, 그래프 위에서 밀접하게 연결된 노드들의 집합을 찾아내는 커뮤니티 탐지(Community Detection)가 필요해집니다. 기본 전제는 단순합니다. 비슷한 노드들은 서로 가깝게 연결되어 있을 것이라는, 일종의 유유상종 가정입니다. 이 가정이 얼마나 잘 들어맞는지를 재는 척도가 Modularity로, 같은 커뮤니티 안의 연결은 많고 커뮤니티 밖으로 나가는 연결은 적을수록 값이 높아집니다.

가장 널리 쓰이는 방법론은 2008년 제안된 Louvain 알고리즘입니다. 각 노드를 Modularity가 최대화되는 커뮤니티로 옮기는 Local Moving과, 그렇게 찾은 커뮤니티를 하나의 노드로 압축해 새 그래프를 만드는 Aggregation, 이 두 단계를 반복합니다. 더 이상 노드가 이동하지 않거나 전체 Modularity가 늘지 않거나 그래프가 줄어들지 않으면 멈추고, 결과로 계층적인 커뮤니티 구조를 얻습니다.

![](/images/graphrag/image4.png)

문제는 Louvain이 전체 Modularity를 높이는 데만 집중하다 보니, 같은 커뮤니티로 묶인 노드끼리도 실제로는 연결되어 있지 않은 경우가 생긴다는 점입니다. 이를 보완한 것이 Leiden 알고리즘으로, 모든 커뮤니티가 내부적으로 실제 연결되어 있음을 보장합니다. Local Moving과 Aggregation 사이에 Refinement라는 단계를 새로 끼워 넣는데, 이 단계에서는 커뮤니티 내부의 각 노드를 일단 독립적인 커뮤니티로 취급한 뒤, 실제로 서로 잘 연결된 노드들끼리만 다시 묶고 연결이 없는 노드는 그대로 혼자 남겨둡니다.

![](/images/graphrag/image5.png)

마지막으로 짚어둘 배경 개념은 Sensemaking입니다. 연결 관계를 이해하고 미래를 예측해 효과적으로 행동하기 위한 지속적인 노력을 뜻하는데, 신호를 포착(Noticing)하고, 의미를 부여해 기존 지식과 연결(Interpreting)하고, 그에 기반해 행동(Acting)하고, 결과를 평가해 새로운 이해를 형성(Reflection)하는 과정으로 이뤄집니다. GraphRAG가 지향하는 global sensemaking 능력이 바로 이 개념에서 나옵니다.

## GraphRAG의 파이프라인

![](/images/graphrag/image6.png)

GraphRAG는 크게 인덱싱(Indexing)과 쿼리 시점(Query Time) 두 단계로 나뉩니다. 인덱싱 단계에서는 문서를 chunking하고, 각 청크에서 엔티티와 관계를 추출해 지식 그래프를 구축한 뒤, 그래프 커뮤니티를 탐지하고, 각 커뮤니티에 대한 요약을 미리 만들어 둡니다. 쿼리 시점에는 이 커뮤니티 요약들로부터 각각의 답(local answer)을 생성하고, 이를 종합해 최종적인 글로벌 답변을 만듭니다.

몇 가지 구현 디테일이 흥미롭습니다. 먼저 청크 크기가 커질수록 추출되는 엔티티 개수는 오히려 줄어드는 경향이 있는데, 청크가 길어질수록 중복된 엔티티가 많아지고 관계가 뭉뚱그려져 요약되기 때문으로 보입니다. 엔티티와 관계 추출 자체는 LLM에 few-shot 예시를 포함한 프롬프트로 수행하고, 여기에 더해 날짜나 이벤트, 다른 엔티티와의 상호작용처럼 엔티티에 대한 중요한 사실을 뽑아내는 Claim 추출도 함께 이뤄집니다. 이 과정에는 Self-Reflection, 즉 LLM이 스스로 답을 검토하고 필요하면 다시 생성하도록 유도하는 프롬프팅 기법도 활용되는데, 이를 통해 청크 크기를 키워 호출 횟수를 줄이면서도 탐지되는 엔티티 수는 오히려 늘릴 수 있었다고 합니다.

같은 엔티티나 관계, Claim이라도 청크마다 표현이 조금씩 다를 수 있기 때문에, 동일한 대상에 대한 description들을 모두 모아 하나로 요약해 통일하고, 관계의 등장 횟수를 edge weight로 사용합니다. 이렇게 정리된 그래프에 Leiden 알고리즘으로 two-level 커뮤니티 탐지를 수행합니다.

![](/images/graphrag/image7.png)

각 커뮤니티에 대한 요약(community summary)은 리포트 형식으로 생성됩니다. LLM의 입력 길이 제한을 피하기 위해 커뮤니티에 우선순위를 매겨 순서대로 입력하는데, leaf level 커뮤니티는 내부 edge 수의 합으로, 상위 level 커뮤니티는 하위 커뮤니티 요약문의 길이가 짧은 것부터 순서를 정합니다.

쿼리 시점에는 이렇게 만들어둔 community summary들을 무작위로 섞은 뒤 정해진 길이로 다시 청크를 나눕니다. 각 청크를 바탕으로 개별 답변(local answer)을 생성하면서, 동시에 그 답변이 주어진 질문에 얼마나 도움이 되는지를 100점 만점으로 점수를 매깁니다. 마지막으로 점수가 높은 local answer부터 순서대로, 토큰 한도에 맞을 때까지 프롬프트에 채워 넣어 최종 글로벌 답변을 생성합니다.

![](/images/graphrag/image8.png)

## 평가: LLM-as-a-judge

Query-Focused Summarization 과업의 특성상, 생성된 global sensemaking 질문들에는 정해진 정답(golden standard)이 없습니다. 그래서 논문은 평가 자체도 LLM에 맡깁니다. 먼저 LLM으로 특정 역할(persona)과 그 역할이 수행하고 싶어할 법한 과업을 생성하고, 이를 바탕으로 페르소나별 질문들을 만들어냅니다. 그런 다음 LLM이 채점자가 되어, 답변이 질문의 모든 측면을 얼마나 상세히 다루는지(Comprehensiveness), 서로 다른 관점을 얼마나 풍부하게 담고 있는지(Diversity), 독자가 정보에 기반한 판단을 내리는 데 얼마나 도움이 되는지(Empowerment), 얼마나 간결하고 정확한지(Directness) 네 가지 기준으로 점수를 매깁니다. Comprehensiveness와 Directness는 태생적으로 상충하는 관계에 있는데, 다만 이 네 가지 평가 기준 자체에 대한 명확한 근거나 선행 연구 인용은 논문에서 찾아보기 어려웠습니다.

비교 대상은 세 가지였습니다. GraphRAG의 커뮤니티 탐지 결과를 네 개 레벨(가장 포괄적인 루트 레벨 C0부터 가장 세분화된 C3까지)로 나눠 비교한 것, 커뮤니티 요약 대신 실제 원문 요약을 사용하되 그래프까지는 구축해서 질문 임베딩과 가장 가까운 엔티티를 최대 20개 뽑아 관련 원문을 선택하는 방식(Text Source, TS), 그리고 텍스트 청크를 그대로 벡터화해 질문과 가장 가까운 문서를 찾는 일반적인 RAG였습니다.

Podcast(노드 8,564개, 엣지 20,691개)와 News(노드 15,754개, 엣지 19,520개) 두 데이터셋으로 실험한 결과, Comprehensiveness와 Diversity 기준에서는 GraphRAG가 확연히 앞섰습니다. 반대로 Directness 기준에서는 일반 RAG가 근소하게 우세했는데, 이는 GraphRAG가 여러 커뮤니티의 정보를 종합하다 보니 답변이 상대적으로 덜 간결해지는 경향과 맞닿아 있는 것으로 보입니다. Empowerment 기준에서는 두 방식 사이에 큰 차이가 없었습니다.

![](/images/graphrag/image9.png)

## 마무리

결국 GraphRAG가 잘하는 지점은 명확합니다. 코퍼스 전체를 아울러야 답할 수 있는 질문, 즉 global sensemaking이 필요한 질문에서는 지식 그래프와 커뮤니티 요약을 미리 준비해두는 접근이 확실한 이점을 보입니다. 반대로 짧고 명확한 사실 하나를 빠르게 찾아야 하는 질문이라면, 굳이 그래프를 거치지 않는 일반 RAG가 더 간결한 답을 줄 수도 있습니다. 결국 어떤 질문을 다루려는지가 GraphRAG를 쓸지 말지를 가르는 기준이 되는 셈입니다.

## References
- [From Local to Global: A GraphRAG Approach to Query-Focused Summarization](https://arxiv.org/pdf/2404.16130)
