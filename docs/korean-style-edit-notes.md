# 한글 글 문체 수정 기록

작업일: 2026-09-23

`write-korean-article` 스킬을 기준으로 한글 글 13편을 다듬었습니다. 질문과 예시를 통해 동작을 설명하는 흐름은 유지하고, 어색한 번역투·과장된 표현·문장 연결을 국소적으로 수정했습니다. 기존 소제목과 섹션 순서는 유지했습니다.

## 수정한 문서

- [Accidental Quadratic Hashmap Iteration](../content/posts/accidental-quadratic-hashmap-iteration.ko.md)
- [Community Detection](../content/posts/community-detection.ko.md)
- [Debugging False Sharing](../content/posts/debugging-false-sharing.ko.md)
- [Ego-Splitting Framework](../content/posts/ego-splitting-framework.ko.md)
- [GraphRAG](../content/posts/graphrag.ko.md)
- [KIP-848](../content/posts/kip-848-next-generation-consumer-rebalance-protocol.ko.md)
- [KIP-932](../content/posts/kip-932-queues-for-kafka.ko.md)
- [Kubernetes Topology Aware Routing](../content/posts/kubernetes-topology-aware-routing.ko.md)
- [LightMem](../content/posts/lightmem.ko.md)
- [SIMA](../content/posts/sima-generalist-ai-agent-3d-environments.ko.md)
- [SimpleMem](../content/posts/simple-mem.ko.md)
- [Sparse Matrix & CSR](../content/posts/sparse-matrix-and-csr.ko.md)
- [Spinlock vs Mutex](../content/posts/spinlock-vs-mutex.ko.md)

## 변경 예시

| 이전 표현 | 수정한 표현 |
| --- | --- |
| 이 공백이 특히 두드러지는 지점이 Query-Focused Summarization입니다. | 이러한 한계가 특히 두드러지는 태스크이 Query-Focused Summarization입니다. |
| 장기 기억은 두 가지 서로 다른 리듬으로 관리됩니다. | 장기 기억은 업데이트 시점에 따라 두 가지 방식으로 관리됩니다. |
| 물론 공짜는 없습니다. | 다만 CSR에도 한계가 있습니다. |
| 첫 번째는 MACHINE_CLEAR 빈도가 왜 함께 증가했는가입니다. | 첫 번째 의문은 MACHINE_CLEAR 빈도가 왜 함께 증가했는지입니다. |

## 별도로 확인할 내용

아래는 문체를 읽는 과정에서 발견한 내용상 불일치입니다. 이번 작업에서는 본문의 기술 설명·수치·코드를 수정하지 않았습니다. 논문과 구현에 대한 전체 사실 검증을 수행한 목록은 아닙니다.

### Sparse Matrix & CSR

1. **용량 계산의 단위:** 원소가 `10^12`개인 행렬에 필요한 용량을 `7.3PB`로 설명합니다. 원소당 자료형 크기가 명시되어 있지 않습니다. 가령 원소당 8바이트라면 `8 × 10^12`바이트, 즉 8TB(약 7.3TiB)이므로 본문의 PB 표기는 재확인이 필요합니다.
2. **그래프와 인접 행렬의 불일치:** 그림에는 노드 1과 3 사이의 연결이 없지만, 인접 행렬에는 해당 연결이 1로 표시되어 있습니다. 이후 CSR 예제는 이 행렬을 따르므로 어느 예시가 의도한 것인지 확인해야 합니다.
3. **작은 예시의 저장 공간 비교:** 본문은 4×4 행렬의 16칸과 CSR의 저장 공간을 비교하지만, 실제 예시는 `values` 10개, `column_indices` 10개, `row_pointers` 5개로 구성됩니다. 같은 크기의 원소를 가정하면 이 작은 예시에서는 CSR이 더 많은 공간을 사용합니다. 또한 포인터 개수는 행 개수 자체가 아니라 행 개수에 1을 더한 값입니다. 희소성이 충분히 높을 때의 이점과 이 예시 자체를 구분할 필요가 있습니다.

### LightMem

주제 경계를 두 후보 집합의 교집합 `B = B_1 ∩ B_2`로 정한 뒤, 한쪽 신호가 놓치는 지점을 다른 신호가 보완한다고 설명합니다. 교집합은 양쪽 후보에 모두 포함된 경계만 남기므로, 한쪽에만 있는 후보를 살리는 의미의 설명과는 맞지 않습니다. 원문에서 설명하는 결합 목적을 확인할 필요가 있습니다.

## 보존 및 검증

- SIMD JSON 이전 한글 글 11편과 Flash Attention 한글 글은 수정하지 않았습니다.
- SIMD JSON을 포함한 영어 본문도 수정하지 않았습니다. 작업 전부터 있던 `flash-attention.en.md`의 변경은 그대로 보존했습니다.
- 수정 전 사본과 비교하여 13편의 front matter, 소제목, 코드 블록, 수식, 링크·이미지 경로, 숫자 표기가 동일함을 확인했습니다.
- 한국어 비활성화 설정은 수정하지 않고, 검증용 임시 설정에서 한국어를 활성화하여 Hugo 빌드를 확인했습니다.
- 커밋과 푸시는 수행하지 않았습니다.
