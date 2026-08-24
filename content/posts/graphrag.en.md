+++
title = 'GraphRAG'
date = '2026-03-13T10:26:08+09:00'
draft = false
translationKey = 'graphrag'
slug = 'graphrag-en'
aliases = ['/posts/graphrag-en/']
description = "How GraphRAG combines knowledge graphs with community detection to answer questions that need a view of an entire corpus, walked through its indexing/query pipeline and evaluation results."
tags = ['RAG', 'AI', 'Knowledge Graph', 'LLM']
categories = ['AI', 'RAG']

[cover]
image = 'images/graphrag/image6.png'
hiddenInSingle = true
+++

> This post summarizes the Microsoft Research paper [From Local to Global: A GraphRAG Approach to Query-Focused Summarization](https://arxiv.org/pdf/2404.16130), also drawing on the video [\[Paper Review\] GraphRAG](https://www.youtube.com/watch?v=mlsZIThxQcQ) by Seoul National University's DSBA Lab.

RAG works by building a trusted document collection ahead of time, then, when a question comes in, retrieving the relevant documents and handing them to an LLM as grounding for its answer. The basic pieces are indexing (chunking documents into a searchable form), retrieval (finding documents relevant to the question), and generation (producing an answer from the retrieved documents and the question).

![](/images/graphrag/image1.png)

## Where Regular RAG Falls Short

RAG's progress has mostly centered on two things: understanding a question's intent more precisely, and finding the documents relevant to it more effectively. That focus made single-document retrieval strong, but comparatively little attention went to the relationships and connections between documents.

![](/images/graphrag/image2.png)

That gap shows up clearly in query-focused summarization. Questions like "what's the overall theme of this dataset?" or "what are the key trends across the last decade of research?" can only be answered by taking in the whole corpus, and a handful of retrieved documents can't cover that ground. GraphRAG targets exactly this gap.

## Background: Knowledge Graphs

A knowledge graph represents a knowledge base, triplets describing what kind of thing two objects are and how they relate, as a graph.

![](/images/graphrag/image3.png)

It compresses a lot of information into relatively little text, makes the connections between entities directly visible, and lets you tune the granularity of nodes and relationships as needed.

Knowledge Base Question Answering (KBQA) predates GraphRAG as a way to answer questions using a knowledge graph. On the surface they look similar (both take a text question and return a text answer), but they differ quite a bit underneath. KBQA aims to retrieve a precise entity from a structured knowledge base like Freebase or Wikidata, converting the question into a logical form like SPARQL and reasoning over it with rules. GraphRAG, by contrast, converts a document collection into a graph and combines graph traversal with LLM generation to produce a natural-language answer. In that sense, GraphRAG can be seen as a generalization of KBQA.

The practical way to build a knowledge graph is to feed documents to an LLM and prompt it to extract triplets of entities and relationships. Collecting all the resulting triplets builds out the full graph.

## Background: Community Detection

Once you have a knowledge graph, you need community detection, finding sets of densely connected nodes within it. The underlying assumption is simple: similar nodes tend to cluster together, a kind of birds-of-a-feather effect. Modularity measures how well that assumption holds, rising as connections stay dense within a community and sparse across community boundaries.

The most widely used method is the Louvain algorithm, proposed in 2008. It alternates between two phases: Local Moving, which shifts each node into whichever community maximizes modularity, and Aggregation, which collapses the communities found so far into single nodes to form a new graph. It stops once no node moves, overall modularity stops increasing, or the graph stops shrinking, producing a hierarchical community structure as its output.

![](/images/graphrag/image4.png)

The catch is that Louvain optimizes purely for overall modularity, which can leave nodes grouped into the same community without being connected to each other. The Leiden algorithm fixes this, guaranteeing that every community is internally well-connected. It inserts a new Refinement step between Local Moving and Aggregation: within each community, every node briefly starts out as its own separate community, and only nodes that are well-connected to each other get merged back together, leaving disconnected nodes on their own.

![](/images/graphrag/image5.png)

One more background concept worth pinning down is sensemaking: the sustained effort of understanding connections and predicting the future in order to act effectively. It plays out as a cycle of noticing important signals in the environment, interpreting them by connecting them to existing knowledge, acting on that interpretation, and reflecting on the outcome to build new understanding. The global sensemaking capability GraphRAG aims for comes directly from this idea.

## The GraphRAG Pipeline

![](/images/graphrag/image6.png)

GraphRAG splits into two phases: indexing and query time. During indexing, documents get chunked, entities and relationships get extracted from each chunk to build a knowledge graph, communities get detected in that graph, and a summary gets pre-generated for each community. At query time, each community summary produces its own local answer, and those get synthesized into a final global answer.

A few implementation details stand out. Larger chunk sizes tend to yield fewer extracted entities, likely because longer chunks introduce more duplicate entities and compress relationships into vaguer summaries. Entity and relationship extraction itself runs through an LLM prompt that includes few-shot examples, alongside claim extraction, which pulls out important facts about an entity: dates, events, interactions with other entities. This process also uses self-reflection, prompting the LLM to review its own output and regenerate if needed, which the authors found let them use larger chunks (cutting the number of LLM calls) while still increasing the number of entities detected.

Since the same entity, relationship, or claim can get described slightly differently across chunks, all descriptions of the same thing get collected and merged into a single unified summary, and how often a relationship appears becomes its edge weight. The resulting graph then goes through two-level community detection using the Leiden algorithm.

![](/images/graphrag/image7.png)

Each community's summary is generated as a report. To stay within the LLM's input length limit, communities are ranked and fed in priority order: leaf-level communities by the total number of internal edges, and higher-level communities starting with whichever sub-community summaries are shortest.

At query time, these pre-built community summaries get shuffled and re-chunked to a fixed length. Each chunk produces a local answer, alongside a score, out of 100, for how helpful that answer is to the given question. The final global answer gets built by filling the prompt with the highest-scoring local answers, in order, until it hits the token limit.

![](/images/graphrag/image8.png)

## Evaluation: LLM-as-a-Judge

Given the nature of query-focused summarization, the global sensemaking questions used for evaluation have no golden-standard answer. So the paper hands evaluation itself to an LLM. It first generates a persona and a task that persona would plausibly want to accomplish, then generates questions for each persona based on that. An LLM then acts as judge, scoring answers on four criteria: comprehensiveness (how thoroughly the answer covers every aspect and detail of the question), diversity (how rich and varied the perspectives it offers are), empowerment (how much it helps the reader make an informed judgment on the topic), and directness (how concise and precise it is). Comprehensiveness and directness are naturally in tension with each other, though the paper doesn't offer much justification or prior work backing this particular set of four criteria.

The paper compared three approaches: GraphRAG's community detection results across four levels (from C0, the most comprehensive root level, down to C3, the most granular); a text-source (TS) variant that builds the graph but skips community summaries, instead pulling up to 20 entities whose embeddings are closest to the query and using the original text chunks those entities appear in; and a standard RAG baseline that vectorizes text chunks directly and retrieves whichever is closest to the query embedding.

Tested on two datasets, Podcast (8,564 nodes, 20,691 edges) and News (15,754 nodes, 19,520 edges), GraphRAG came out clearly ahead on comprehensiveness and diversity. Standard RAG edged it out slightly on directness, which tracks with GraphRAG's tendency to produce less concise answers as it synthesizes information across multiple communities. Empowerment showed little difference between the two.

![](/images/graphrag/image9.png)

## Wrap-up

GraphRAG's strength is clear enough: questions that require a global view of the entire corpus, genuine global sensemaking, benefit clearly from pre-building a knowledge graph and community summaries. For a quick, narrow factual lookup, though, a standard RAG pipeline that skips the graph entirely may give a more concise answer. In the end, what kind of question you're trying to answer is what decides whether GraphRAG is worth the overhead.

## References
- [From Local to Global: A GraphRAG Approach to Query-Focused Summarization](https://arxiv.org/pdf/2404.16130)
- [\[Paper Review\] GraphRAG (Seoul National University DSBA Lab)](https://www.youtube.com/watch?v=mlsZIThxQcQ)
