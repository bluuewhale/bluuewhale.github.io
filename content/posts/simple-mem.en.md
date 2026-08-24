+++
date = '2026-01-24T17:56:58+09:00'
draft = false
math = true
translationKey = 'simple-mem'
title = '[Paper Review] SimpleMem: Efficient Lifelong Memory for LLM Agents'
slug = 'simple-mem-en'
aliases = ['/posts/simple-mem-en/']
categories = ['RAG', 'AI']
tags = ['RAG', 'AI']
description = 'A review of the SimpleMem paper, covering semantic compression, recursive memory consolidation, adaptive retrieval, and benchmark results.'
keywords = ['SimpleMem', 'LLM Memory', 'RAG', 'Semantic Compression', 'Adaptive Retrieval', 'Long Context']

# SEO / Social
# - PaperMod uses .Description for meta description + OG/Twitter/Schema. If omitted, it falls back to .Summary.

# PaperMod reads .Params.cover.image for:
# - post cover rendering
# - OG/Twitter image (highest priority)

[cover]
image = 'images/simple-mem/pipeline.png'
alt = 'Diagram of the 3-stage SimpleMem pipeline'
caption = 'SimpleMem Architecture: Semantic Structured Compression, Online Semantic Synthesis, Intent-Aware Retrieval Planning'
relative = false
hiddenInSingle = true
+++

> This post is a review of [*SimpleMem: Efficient Lifelong Memory for LLM Agents*](https://arxiv.org/pdf/2601.02553).
>
> Some parts of the paper have been updated after this post was written (2026-01-24), so there may be differences from the version discussed here.

## Background
LLMs are stateless. As a result, previous inference outputs do not directly affect later outputs.

Because of this property, a plain LLM can fail to maintain continuity in long conversations. In other words, it may look like short-term memory loss, where it cannot remember what was just discussed.

Researchers addressed this with a straightforward idea: store conversation history between the user and the LLM agent in a separate memory space, then inject relevant history into the prompt at each inference step so the model can sustain continuity.

This approach is intuitive and effective, but it introduces context-length growth. Adding historical exchanges to the prompt makes input length grow.

Longer prompts cause secondary issues. First, prompt length can exceed the model's hard context limit, making inference impossible.

In addition, longer context increases latency and can hurt quality. According to [Databricks research](https://www.databricks.com/blog/long-context-rag-performance-llms), excessive context can degrade RAG performance.

Their experiments show that quality improves up to a point as they add more supporting context, but beyond that point quality plateaus or even drops. This pattern appears consistently across different LLM families.

![Databricks Research Result](/images/simple-mem/databricks.webp)

As context grows too large, models may show abnormal behaviors: confidently giving incorrect answers (GPT-4), refusing to answer with unrelated copyright concerns (Claude-3-Sonnet), or repeating the same tokens endlessly (Mixtral-instruct).

![Mixtral-instruct answering repeated content](/images/simple-mem/mixtral.png)

Memory systems face a similar limitation. As interactions continue, the prompt grows, token usage rises, and performance can drop. The effect compounds over long sessions.

The paper proposes a semantic lossless compression framework to address these issues.

## Keywords
- Entropy-aware filtering
- Recursive Memory Consolidation
- Adaptive Query-Aware Retrieval

## Proposed Method

The paper introduces a three-stage pipeline:

![pipeline](/images/simple-mem/pipeline.png)

### 1. Semantic Structured Compression
> Discard unimportant dialogue and keep only informative content.

The paper argues that the main bottleneck in long-term conversational memory is context inflation: too much low-value content accumulates over time. To address this, it proposes a filtering process that scores the informativeness of each window using entropy-inspired signals.

**Information score**

To separate important dialogue from unimportant dialogue, the method estimates an information score. The paper defines it as:

![entrophy](/images/simple-mem/entrophy.png)

The information score ($H(W_t)$) combines the following terms:

- $\frac{|\mathcal{E}_{new}|}{|W_t|}$: entity-level novelty
  - $|\mathcal{E}_{new}|$: the number of entities not seen in accumulated memory ($H_{prev}$)
  - $|W_t|$: the dialogue window length (overlapping sliding window size)
  - In short, this term measures how much new entity-level information appears in the current window.
- $1 - cos(E(W_t),E(H_{prev}))$: semantic divergence
  - $E(W_t)$: embedding of the current dialogue window
  - $E(H_{prev})$: embedding of past memory
  - If the two are semantically similar, cosine similarity increases and this divergence term decreases.
- $\alpha$: weighting factor between entity novelty and semantic divergence

**Filtering**

The next step applies threshold-based filtering using the information score. If the score falls below a threshold ($\tau_{redundant}$), the method treats the window as non-informative and drops it ($\varnothing$).

![filtering](/images/simple-mem/filtering.png)

**Memory unit ($m_k$) construction**

From the filtered dialogue, the system constructs memory units.

![memory unit](/images/simple-mem/memory-unit.png)

A memory unit is a context-grounded and reusable representation of dialogue content. Construction consists of three transformations:

- $\Phi_{extract}(W_t)$: extract factual statements from dialogue
- $\Phi_{coref}$: resolve pronouns into concrete entities (e.g., `He agreed` -> `Bob agreed`)
- $\Phi_{time}$: convert relative time into absolute ISO-8601 timestamps (e.g., `next Friday` -> `2025-10-24`)

### 2. Structured Indexing & Recursive Memory Consolidation
> Index memory in multiple ways and periodically consolidate it for efficiency.

#### Structured indexing

The system stores each memory unit through three complementary index views:

![structured indexing](/images/simple-mem/structured-indexing.png)

- Semantic Layer: dense vector embedding for semantic matching
- Lexical Layer: inverted-index style keyword retrieval
- Symbolic Layer: key-value metadata tags for structured constraints

This allows the system to handle diverse query patterns (semantic, keyword, and metadata-based search) more robustly.

#### Recursive memory consolidation

Even with filtering, memory can still accumulate excessively over time. To control this, the paper proposes an asynchronous background consolidation process that merges similar memory units.

First, for stored memory units, it computes an affinity score ($w_{ij}$) between two units ($m_i$, $m_j$):

![affinity score](/images/simple-mem/affinity-score.png)

The affinity score combines:
- $cos(v_i,v_j)$: semantic relatedness between two memory units
- $e^{-\lambda |t_i - t_j|}$: temporal proximity between two memory units

Then the system consolidates similar units:

![memory consolidation](/images/simple-mem/memory-consolidation.png)

In this step, the method groups units with affinity above a threshold ($C$), and an LLM-based synthesis function ($G_{syn}$) merges them into abstract memory units. The final output is an abstracted unit $M_{abs}$.

For example, if there are many memories about ordering coffee at 8 a.m., the system can compress them into an abstract unit like "the user usually drinks coffee in the morning," preventing uncontrolled memory growth.

This idea of merging small episodic memories into a more abstract representation also appears in prior work. A representative example is [Generative Agents](https://arxiv.org/pdf/2304.03442), which uses a similar process called reflection.

![reflection](/images/simple-mem/reflection.png)

### 3. Adaptive Query-Aware Retrieval
> Use lightweight recall for simple questions, and broader multi-view retrieval for complex ones.

The final stage retrieves memory units for response generation.

Standard RAG systems usually fetch top-k content at a fixed retrieval depth. This paper points out that the number of units needed varies by query complexity. It therefore proposes a query layer that dynamically adjusts retrieval depth.

#### Hybrid Scoring Function

The paper defines a relevance score $S(q, m_k)$ between query ($q$) and memory unit ($m_k$):

![hybrid-scoring-function](/images/simple-mem/hybrid-scoring-function.png)

- $\lambda_1 \cos(e_q, v_k)$: semantic similarity between query and memory unit
- $\lambda_2 \text{BM25}(q_{lex}, S_k)$: lexical overlap between query keywords and the memory unit
- $\gamma I(R_k \models C_{meta})$: whether memory tags ($R_k$) satisfy query constraints ($C_{meta}$)

Finally, the system dynamically adjusts the number of recalled units ($k_{dyn}$) based on query complexity:

![adaptive-k-retrieval](/images/simple-mem/adaptive-k-retrieval.png)

- $k_{base}$: minimum memory count used for answer generation
- $C_q$: query complexity in [0, 1], estimated by a lightweight classifier
- Higher complexity ($C_q$) leads to larger retrieval depth

## Experiments

The paper claims that, when considering both effectiveness (F1) and efficiency (inference token cost), SimpleMem achieves a better trade-off than prior memory approaches.

### Benchmarks

The evaluation focuses on LoCoMo and LongMemEval-S.

LoCoMo is a long-context conversational benchmark with 200-400 turns and 1,986 evaluation questions. LongMemEval-S tests precise retrieval over extremely long interaction histories. For LongMemEval-S, the paper uses an LLM-as-a-judge protocol with gpt-4.1-mini, labeling outputs as CORRECT or WRONG.

### Results

The authors report an average F1 improvement of 26.4% on LoCoMo and up to 30x lower inference-time token usage compared to full-context baselines. They also report stronger performance than the previous SOTA memory framework [Mem0](https://github.com/mem0ai/mem0).

![result](/images/simple-mem/result.png)
