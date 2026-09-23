+++
date = '2026-09-18T13:14:49+09:00'
draft = false
title = 'FlashAttention'
math = true
translationKey = 'flash-attention'
slug = 'flash-attention'

# SEO / Social
# - PaperMod uses .Description for meta description + OG/Twitter/Schema. If omitted, it falls back to .Summary.
description = 'FlashAttention이 GPU 메모리 I/O 병목을 줄이는 원리를 살펴봅니다. 타일링과 online softmax의 보정 수식, 알고리즘, I/O 복잡도, block-sparse 확장과 실험 결과를 정리합니다.'
tags = ['AI', 'LLM', 'Transformer', 'Attention', 'GPU', 'FlashAttention']
categories = ['AI', 'LLM']
keywords = ['FlashAttention', '플래시 어텐션', 'Tiling', 'Online Softmax', 'GPU Memory', 'HBM', 'SRAM', 'IO-aware Attention', 'Block-Sparse Attention', 'Sparse Transformer', 'Pixelated Butterfly']

# PaperMod reads .Params.cover.image for:
# - post cover rendering
# - OG/Twitter image (highest priority)
[cover]
image = 'images/flash-attention/image16.png'
alt = 'FlashAttention의 query 행 블록과 key 열 블록, 행별 누적 출력 및 정규화 통계량을 나타낸 구조도'
caption = 'FlashAttention의 타일링 구조와 행별 누적 상태'
relative = false
hiddenInSingle = true
+++
# 배경

## 기존 어텐션 알고리즘은 잦은 HBM를 유발하여 I/O 병목을 발생시킨다.

저자는 어텐션 연산 과정에서 발생하는 I/O 병목에 주목하며, 어텐션 연산 과정에서  과도하게 HBM에 접근하는 기존 어텐션 알고리즘의 I/O 패턴을 핵심 원인으로 지목하였습니다. 이 현상을 효과적으로 이해하기 위해서는 먼저, GPU를 포함한 대부분의 가속기가 갖고 있는 계층화된 메모리 구조에 대해서 먼저 이해해야 합니다.

## 이론적인 연산 복잡도(FLOPS)가 아니라 실질적인 추론 시간(wall-clock)을 개선한다.

저자는 선행 연구들에서 clock 어텐션을 계산하는 과정의 연산/메모리 복잡도를 낮추기 위해 노력하였지만, 많은 연구에서 실질적인 wall-clock 개선을 리포트 하고 있지 않다고 지적합니다. 이는 선행 연구들이 주로 연산의 이론적인 FLOP를 줄이는데 초점을 두고, 메모리 접근에 의해 발생하는 오버헤드를 무시하고 있기 때문이라고 이야기합니다.

저자는 어텐션 연산 과정의 실질적인 wall-clock 시간을 줄이기 위해서는 어텐션 연산 과정에서 발생하는 메모리 병목에 주목해야 한다고 주장합니다. 이 부분이 논문의 핵심 주장이라고 볼 수 있는데요. 저자는 어텐션 연산을 IO-aware 하게 재설계하여 SRAM-HBM 사이의 데이터 교환을 최소화함으로써  어텐션 연산을 가속하는 FlashAttention 알고리즘을 제안합니다.

## 메모리 계층 구조

GPU의 메모리는 연산 칩에 on-chip SRAM과 GPU의 메인 메모리 역할을 하는 HBM으로 구성됩니다. SRAM은 GPU 내부의 SM(streaming multiprocessor)들이 고유하게 보유한 메모리 캐시로 대역폭이 매우 높지만, 용량이 매우 작습니다. 반면, HBM은 상대적으로 용량이 크지만, 대역폭 면에서는 SRAM에 크게 못 미치는 수치를 보여줍니다. GPU 보다는 상대적으로 조금 더 친숙한 CPU에 비유하자면, SRAM은 CPU 칩에 위치한 캐시 계층(L1~L3)이라고 할 수있고, HBM은 CPU 소켓에 연결된 DRAM이라고 이야기할 수 있을 것 같습니다.

(SM을 CPU로 치면 코어로 비유할 수 있을지 확인하기)

![](/images/flash-attention/image1.png)

## 어텐션 연산의 딜레마

논문에서 직면한 trade-off는 컴퓨팅 공학 분야에서 매우 빈번하게 등장하는 단골 주제라고 할 수 있습니다. SRAM은 빠르지만 작고, HBM은 크지만 느립니다. 단순하게 생각하면, 연산 과정에서 느린 HBM에 덜 접근 할수록 I/O 오버헤드는 작아집니다. 하지만, SRAM은 어텐션 연산에서 발생하는 중간 산출값들을 모두 담기에는 너무 작으므로  임시로 HBM에 값을 담아둘 수 밖에 없습니다.

## 어텐션 알고리즘의 동작 방식

저자는 이 딜레마를 해결하기 위해 실제 어텐션 연산이 이뤄지는 과정에 주목합니다.  일반적으로 길이 $N$의 주어진 입력 토큰 시퀀스($Q$)에 대한 어텐션 출력($O$)은 다음과 같이 계산됩니다. 여기서 주목할 부분은 연산 과정에서 만들어지는 중간 산출물인 $S$ 와 $P$가 입력 시퀀스 길이의 제곱($N^2$)에 해당하는 $(N, N)$ 매우 큰 크기의 행렬이라는 점입니다. 논문에서는 이렇게 거대한 $S$, $P$ 행렬을 읽고 쓰는 과정에서 발생하는 I/O 오버헤드가 어텐션 연산을 느리게 만드는 주범이라고 지목합니다.

$S$, $P$ 행렬이 유발하는 메모리 I/O 병목을 더 자세히 이해하기 위해, 어텐션 알고리즘의 동작 방식을 차근차근 순서대로 살펴보도록 하겠습니다. 일반적인 어텐션 알고리즘은 다음과 같이 동작합니다.

1. HBM으로부터 $Q$,$K$ 블록을 읽어와서 SRAM에 복사한 후, $S$를 계산($S = QK^T$)한 후, 결과 값을 HBM에 저장합니다.
2. HBM으로부터 다시 $S$를 읽어와서, 행 단위로 softmax 연산을 적용하여$P$를 계산($P = softmax(S)$)한 후, 결과 값을 다시 HBM에 저장합니다.
3. HBM으로부터 다시 $P$와 $V$ 블록을 읽어온 후, 최종 어텐션 출력(O)을 계산($O = PV$)한 후, 결과 값($O$)을 HBM에 저장합니다.

![](/images/flash-attention/image3.png)

$$
S = \frac{QK^{\mathsf{T}}}{\sqrt{d}}, \qquad
P = \operatorname{softmax}_{\mathrm{row}}(S), \qquad
O = PV
$$

$$
Q, K, V, O \in \mathbb{R}^{N \times d}, \qquad
S, P \in \mathbb{R}^{N \times N}
$$

![](/images/flash-attention/image2.png)

위 과정을 거치면, 점수 행렬($S$)를 HBM에 저장한 뒤 softmax 계산을 위해 다시 읽고, 그 결과인 가중치 행렬($P$)도 HBM에 저장했다가 최종 출력 계산을 위해 다시 읽게 됩니다. 두 행렬의 크기는 각각 $N \times N$이므로, 입력 길이가 길어질수록 이러한 메모리 I/O 비용도 커집니다. 그렇다면, 중간 결과를 HBM에 저장하지 않고 다음 연산에 바로 활용할 수는 없을까요? 이쯤 되면, 슬슬 논문에서 말하고자 하는 바를 유추 해볼 수 있습니다. 

> 어텐션 연산 과정에서 거대한 중간 값 행렬($S$, $P$)을 HBM에 복사하지 않고도 정확한 어텐션 출력($O$)값을 계산할 수 있는 알고리즘을 제안하겠다!

여기까지 알았으니, 이제부터는 논문에서 어떻게 거대한 중간 산출물 행렬($S$, $P$)을 HBM에 옮겨 담지 않고도 정확한 어텐션 출력($O$)을 계산하는지 구체적인 방법론에 대해 집중적으로 살펴보도록 하겠습니다.

# Flash Attention

논문의 목표는 명확합니다. 

> 주어진 $Q, K, V$에 대해서, HBM 접근을 최소화하면서 정확한 어텐션 출력($O$)을 계산하자!

이 목표를 달성하기 위해 논문에서 제안하는 방법론의 핵심은 "$Q, K, V$를 블록(block) 단위로 쪼개어서 블록 단위로 최종 어텐션 값($O$)을 계산함으로써 불필요하게 중간 산출물($S$, $P$)을 HBM에 저장하는 과정을 제거하자"입니다.

## Tiling

어텐션 값을 계산하는 과정에서 중간 산출물 행렬 $P$를 계산하기 위해서는 또 다른 중산 산출물 행렬 $S$에 대해서 soft softmax 연산을 적용해야 합니다. 

$$
P = softmax(S)
$$

논문에서는 $Q$, $K$, $V$ 행렬을 일정 크기의 블록으로 쪼개고, 블록 단위로 최종 어텐션 레이어 출력($O$)을 계산함으로써, 중간 계산 결과($S$, $P$)를 HBM에 저장하는 메모리 I/O 과정을 제거하고자 하였습니다. 

하지만, 여기에는 한 가지 문제가 있습니다. 바로, 어텐션 출력 계산에 활용되는 softmax 연산을 블록 단위로 적용할 경우, 전체에 한번에 적용한 결과와 일치하지 않는다는 점입니다. softmax 연산은 주어진 벡터의 개별 원소에 지수함수를 적용한 후, 전체 벡터의 변환 값의 합으로 나누어 정규화함으로써 벡터의 모든 원소가 양수이며 전체 벡터의 합이 1이 되는 확률 분포 형태로 변환하는 연산입니다. 다만, 실제 논문에서는 지수함수 적용 시 overflow를 방지하기 위해 max 값을 정규화가 적용되어 있습니다.

$$
\operatorname{softmax}(x)_i = \frac{e^{x_i}}{\sum_{j=1}^{n} e^{x_j}}
$$

![](/images/flash-attention/image4.png)

올바른 softmax 값을 계산하기 위해서는 전체 벡터의 원소에 지수 함수를 적용하여 그 합을 계산해야 합니다. 하지만, 논문에서는 블록 단위로 softmax 값을 계산하고 이에 적절한 보정 계수를 곱 함으로써 전체 벡터로부터 계산한 softmax 값과 동일한 값을 얻을 수 있는 방법론을 제안합니다. 어떻게 벡터의 부분의 합으로 정규화한 결과가, 전체 합으로 정규화한 결과와 일치할 수 있을까요? 이를 이해하기 위해서 논문에서 제안한 방법론을 더 깊이 살펴보도록 하겠습니다.

논문에서 제안하는 방법론의 핵심은 지수 함수의 곱연산의 특성을 활용하여 보정 계수로 보정하여 올바른 값을 계산하는 것입니다.

![](/images/flash-attention/image5.png)

지수 함수의 곱은 지수항의 합으로 계산될 수 있습니다.

$$
e^{a+b} = e^a \cdot e^b
$$

이 특성을 이해한다면, 아래의 수식이 성립하는 것을 쉽게 이해할 수 있습니다. 

논문에서는 블록 단위로 계산한 결과값에 적절한 보정 계수를 곱함으로써 전체 기준으로 계산한 것과 동일한 결과값을 계산합니다.

$$
\underbrace{e^{x_i^{(1)}-m(x)}}_{\text{전체 기준 지수값}}
= \underbrace{e^{x_i^{(1)}-m(x^{(1)})}}_{\text{블록 기준 지수값}}
\cdot \underbrace{e^{m(x^{(1)})-m(x)}}_{\text{보정 계수}}
$$

보정 계수를 계산하기 위해서는 $m(x)$와 $\ell(x)$ 값을 추적할 수 있어야 하므로, 논문에서는 이 값을 별도의 공간에  저장합니다.

# Algorithm

논문의 핵심 알고리즘은 다음과 같습니다.

![](/images/flash-attention/image6.png)



다음으로 논문의 핵심 알고리즘을 한 줄 씩 순서대로 자세히 살펴보도록 합시다.

## Line 1

![](/images/flash-attention/image7.png)

이 단계는 하나의 블록 계산에 필요한 데이터들을 SRAM에 모두 담을 수 있도록 효과적인 블록 크기를 설정하는 단계입니다. 이와 관련하여서는 Appendix C의 $Proof\ of\  Theorm\ 2$ 섹션에서 자세히 설명하고 있습니다. 

![](/images/flash-attention/image8.png)

![](/images/flash-attention/image9.png)

우선, 하나의 블록을 처리하기 위해 필요한 데이터의 크기를 대략적으로 다음과 같이 추정할 수 있습니다.
- $K_j$, $V_j$: 각각 $B_c \times d$ 크기
- $Q_i$, $O_i$: 각각 $B_r \times d$ 크기
- $S_{ij}$: $B_r \times B_c$ 크기

즉, 어텐션 헤드의 차원의 수($d$)가 커지거나 블록의 행 길이($T_r$) 혹은 열 길이($T_c$)가 길어질수록 하나의 블록을 계산하는데 소요되는 메모리 공간의 크기가 증가합니다. 하지만, 하나의 블록을 계산하기 위해 필요한 데이터들은 모두 하나의 SRAM에 담을 수 있어야 합니다. 다시 말해, 어텐션 헤드의 차원($d$), 블록의 행 길이($T_r$) 열 길이($T_c$)는 SRAM의 크기($M$)에 의해 제한된다고 이해할 수 있습니다. 여기서, 어텐션 헤드의 차원($d$)과 SRAM의 크기($M$)는 일반적으로 조정할 수 없는 값이므로, 이 두 값을 반영하여 적절하게 행의 길이와($T_r$) 열의 길이($T_c$)를 선정해야 합니다.이러한 관점에서 하나의 어텐션 블록의 최대 행 길이($T_r$)와 열 길이($T_c$)는 SRAM의 크기($M$)에 비례하고, 어텐션 헤드의 차원의 크기($d$)에 반비례한다고 정의할 수 있습니다. 

아쉽게도 상수 항($4$)에 대한 근거는 찾을 수 없었으며, 실제로는 모델의 $Q$, $K$, $V$의 자료형(ex, $FP32$)등을 고려하여 적절한 블록 크기를 산정해야 할 것으로 보입니다.


## Line 2

![](/images/flash-attention/image10.png)

다음으로, 블록 단위 계산 결과를 저장하기 위한 공간을 할당합니다. 우선, 어텐션 출력 결과($O$)를 계산하기 위한 $N \times d$ 크기의 행렬을 선언합니다. 그리고, 행 단위로 블록 별 계산 결과를 보정하기 위한 $N$ 크기의 $\ell$과 $m$ 벡터를 선언합니다.

## Line 3-4

![](/images/flash-attention/image11.png)

다음으로 실제 블록을 나누는 작업을 진행합니다. 논문에서는 $Q$ 행렬은 $T_r$개씩 $B_r \times d$ 크기의 $T_r$개의 블록으로 나누고, $K$, $V$ 행렬은 $T_c$ 개씩 $B_c \times d$ 크기의 블록으로 나누었습니다. 최종적으로 하나의 블록에 대한 점수 행렬의 크기($S_{ij}$)는 $B_r \times B_c$ 크기가 됩니다.

이와 동일하게, 어텐션 출력 행렬($O$)과 보정 연산에 필요한 값($l$, $m$)도 모두 각각 $B_r$만큼의 길이를 갖는 $T_r$ 개의 벡터로 쪼개었습니다.

![](/images/flash-attention/image12.png)



## Line 5-6

![](/images/flash-attention/image13.png)

모든 블록에 대한 어텐션 출력을 계산하기 위해서는 $T_r$개의 행과 $T_c$개의 열을 모두 순회해야 합니다.

이를 위해, 알고리즘은 이중 루프로 구성되어 있습니다.  논문에서는 점수 행렬 기준($S$)을 채워 나갈 때, 열을 우선으로 채워가도록 ($1 ... T_c$) 루프를 구성하였습니다. Line 5-6은 그 중에서도 바깥 루프(outer loop)를 구성하는 부분에 해당하며 어텐션 키($K_j$), 값($V_j$) 블록을 읽어옵니다.

한 가지 흥미로운 사실은 이러한 결정이 후속 연구에서 번복된다는 사실인데요. 저자는 후속 연구인 Flash Attention 2에서 현 논문의 결정을 번복하고 행 단위로 먼저 순회하는 방식을 제안하고 있습니다. 이에 대해서는 후속 논문을 리뷰하는 과정에서 더 자세히 살펴보도록 하고 우선은 넘어가도록 하겠습니다.

## Line 7-9

![](/images/flash-attention/image14.png)

다음으로 쿼리 행렬($Q$)을 블록 단위($T_r$)로 순회하며 점수 행렬($S$)과 어텐션 출력 행렬($O$)을 계산하는 과정입니다.

이를 위해, 우선 $Q_i$, $O_i$, $l_i$, $m_i$ 값을 HBM으로부터 읽어옵니다.

그런 다음, 점수 행렬의 블록 값($S_{ij}$)을 계산합니다. 

이 과정을 이해하기 쉽게 그림으로 표현해보았습니다.

![](/images/flash-attention/image15.png)

## Line 10-13

![](/images/flash-attention/image16.png)

이 다음 과정이 이 논문의 핵심이라고 할 수 있는 tiling과 recompute 기법이 적용되는 구간입니다. 먼저, 앞서 계산한 점수 행렬 블록($S_{ij}$)에 대해 행 별 최대 값($\tilde{m}_{ij}$)을 계산합니다. 다음으로, softmax 연산에서 각각 분자($f(x)$)와 분모($\ell(x)$)에 해당하는 $\tilde{P}_{ij}$와 $\tilde{l}_{ij}$를 계산합니다. 

이렇게 계산된 값은 해당 열($j$) 블록에 대해서 부분적으로 계산된 값으로, 전체 행에 대한 온전한 softmax 값을 계산하기 위해서는 보정 작업이 필요합니다. 빠르게 다시 복습해보자면, 바로 이 부분입니다.

![](/images/flash-attention/image5.png)

$$
\underbrace{e^{x_i^{(1)}-m(x)}}_{\text{전체 기준 지수값}}
= \underbrace{e^{x_i^{(1)}-m(x^{(1)})}}_{\text{블록 기준 지수값}}
\cdot \underbrace{e^{m(x^{(1)})-m(x)}}_{\text{보정 계수}}
$$

이를 위해, 새로운 $m_i^{new}$와 $l_i^{new}$를 계산합니다.

![](/images/flash-attention/image17.png)

다음으로, 기존에 계산했던 어텐션 출력에 대한 보정 작업을 진행합니다. 논문에서는 softmax 연산을 적용하여 가중치 행렬($P$)을 계산하는 과정과 어텐션 출력($O$)을 계산하는 과정을 분리하지 않고 한번에 처리합니다. 이로 인해, 수식이 다소 복잡하게 보일 수는 있지만 핵심은 간단합니다.

1. 이전까지 누적된 어텐션 출력 행렬을 정규화 이전 상태로 복원합니다. ($diag(l_i)e^{m_i-m_i^{new}}O_i$)
2. 현재 블록에서 계산된 기여도를 반영합니다. ($e^{\tilde{m}_{ij}-m_i^{new}}\tilde{P}_{ij}V_j$)
3. 갱신된 누적 지수 합으로 나눠서 다시 정규화합니다. ($diag(l_i^{new})^{-1}$)

마지막으로 연산 결과($O_i$, $l_i$, $m_i$)를 HBM에 저장합니다. 알고리즘 상에서 확인할 수 있듯, 어텐션 출력($O_i$)을 제외하고 최소한의 통계량($l_i$, $m_i$)만 HBM에 저장됩니다.  중간 산출물에 해당하는 점수 행렬과($S$), 가중치 행렬($P$)은 SRAM 수준에서 버려지게됩니다. 바로 이 부분이 논문의 핵심 기여라고 볼 수 있습니다.

![](/images/flash-attention/image18.png)

이 과정을 $T_c$번 반복하면, 하나의 행에 대해서 온전한 어텐션 출력($O_i$)를 계산할 수 있습니다. 

![](/images/flash-attention/image19.png)

# I/O Complexity of Flash Attention

다음으로 저자는 기존 어텐션과 논문에서 제안하는 Flash Attention의 I/O 복잡도를 비교합니다. 

![](/images/flash-attention/image20.png)

Appendix에서 Flash Attention의 I/O 복잡도에 대한 산출 근거를 자세히 다루고 있습니다. Flash Attention 알고리즘의 이중 루프 구조를 분해해보면, 바깥 루프를 $T_c$번 순회하는데, 내부 루프($i..T_r$)에서는 전체 쿼리 행렬($Q$)과 어텐션 출력 행렬($O$)을 모두 순회합니다. 이 행렬들의 크기는 $N \times d$이므로, Flash Attention 알고리즘의 전체 I/O 복잡도는 $NdT_c$가 됩니다. 마지막으로 $T_c$를 앞선 증명에 따라서 $\frac{Nd}{M}$로 치환하면, 최종적으로 Flash Attention 알고리즘의 I/O 복잡도가 $\frac{N^2d^2}{M}$에 비례함을 계산할 수 있습니다.

![](/images/flash-attention/image21.png)

일반적으로 어텐션 헤드의 차원의 크기($d$)는 64-128로 작은 값을 갖는 반면, SRAM의 크기($M$)는 100KB 수준으로 상대적으로 큰 값을 갖습니다. 따라서, 물론 단위가 달라서 직접적으로 비교할 수는 없지만 대략적으로 $d^2$ &lt; $M$ 이므로 기존 어텐션 대비($Nd + N^2$) I/O 효율이 개선된다고 논문에서 설명하고 있습니다. 

# Block-Sparse Flash Attention

논문에서는 여기서 한 단계 더 나아간 block-sparse flash attention 매커니즘(Algorithm 5)을 제안합니다. 

block-sparse flash attention에서는 전체 블록 개수 만큼의 마스크 행렬($\mathbf{M} \in \{0,1\}^{T_r \times T_c}$)을 활용합니다. 마스크 행렬 중 특정 원소($M_{ij}$)의 값이 0인 경우에는 해당 위치에 대응하는 어텐션 점수 블록($S_{ij}$) 계산을 과감하게 생략합니다. 알고리즘 상에서는 8번 라인에 해당합니다.

![](/images/flash-attention/image22.png)

한 가지 신기한 점은 어떤 블록을 버릴 것인지에  사전에 미리 모두 결정되어 있다는 점인데요. 논문에서는 butterfly sparsity pattern에 따라서 고정된 마스크 행렬을 사용한다고 말하고 있습니다. 어떤 블록을 버릴 것인지 사전에 결정되어 있다는 점이 익숙하지 않게 다가와서 이와 관련하여 추가로 리서치 해보기로 하였습니다.

# 부록: Sparse Transformer &amp; Pixelated Butterfly

논문에서는 Butterfly Sparsity Pattern을 소개하기 위해 Dao et al.의 [Pixelated Butterfly: Simple and Efficient Sparse Training for Neural Network Models (ICLR 2022)](https://arxiv.org/abs/2112.00029)을 직접적으로 인용하고 있습니다. 그리고, 관련 연구들을 찾아보다가 살펴보다가 희소 어텐션(sparse attention)의 초기 대표 연구 격에 해당하는 Child et al. [Sparse Transformer(2019)](https://arxiv.org/abs/1904.10509)을 찾을 수 있었습니다. 이번 글의  주제는 Flash Attention인 만큼 희소 어텐션 관련 내용에 대해서는 되도록 짧게 살펴보고 넘어가도록 하겠습니다.

## Sparse Attention

논문에서 저자는 픽셀 단위로 이미지를 생성하는 자기 회귀 트랜스포머 모델에서 어텐션 가중치 행렬의 독특한 패턴을 관찰하였습니다. 구체적으로 논문에서는 128-layer의 self-attention 네트워크를 CIFAR-10 데이터셋에 적용할 때 등장하는 어텐션 패턴을 분석하였고, 그 결과 4가지 독특한 어텐션 패턴을 발견하였습니다.

- a) 초기 레이어에서는 대부분 다음에 생성할 픽셀과 인접한 위치에 어텐션이 집중되는 모습을 확인할 수 있었습니다. 
- b) 19-20번째 레이어에서는 현재 픽셀과 수직/수평선에 위치한 픽셀들에 어텐션이 집중되는 특이한 모습을 보였습니다.
- c) 일부 어텐션 레이어에서는 이미지 전체 영역에 걸쳐서 다양하게 어텐션 가중치가 분배되는 모습을 보였습니다.
- d) 마지막으로 상대적으로 깊은 레이어(64-128)에서는 특정 입력 패턴에 대해 극히 일부 어텐션만 활성화되는 모습을 보였습니다.

![](/images/flash-attention/image23.png)

이러한 관찰을 기반으로 저자들은 두 가지 어텐션 패턴(Strided, Fixed)을 사전에 정의하고, 해당 패턴에 포함된 영역에 대해서만 어텐션 가중치를 계산하는 희소 어텐션 방법론을 제안하였습니다. 

먼저, Strided 패턴은 인접한 K개의 위치와 J만큼의 간격(stride) 단위로 떨어져있는 위치들을 참조하는 어텐션 패턴입니다. 논문에서는 이미지나 오디오처럼 데이터 자체가 일정 간격(stride)으로 반복되는 구조화된 패턴을 가진 경우에 Stride 어텐션 패턴이 효과적이라고 주장합니다. 다만, 텍스트와 같이 구조화된 패턴이 없는 경우에는 적합하지 않다고 말합니다.

논문에서는 이러한 경우에 대비해서 Fixed 패턴을 제안하였습니다. Fixed 패턴은 전체 입력을 N개의 크기의 cell로 나누고, 과거 cell의 마지막 c개의 입력에 대해서는 이후 모든 어텐션 출력에서 참조하도록 하는 패턴입니다. 예를 들어, cell의 크기를 128, c를 8로 가정하면 매 128개의 입력마다 마지막 8개 입력(120-128)에 대해서는 어텐션을 활성화하는 방식입니다.

![](/images/flash-attention/image24.png)

논문에서는 희소 어텐션 방식이 기존의 밀집(dense) 어텐션 방식 수준의 성능을 유지하며 학습 시간을 크게 단축시킬 수 있음을 보였습니다. 텍스트로 구성된 Enwik8 벤치마크에서는 Fixed 어텐션 패턴을 적용한 희소 트랜스포머 모델이 기존 방식보다 더 좋은 성능을 보이면서, 학습 반복당 소요되는 시간은 1.31 -&gt; 0.55로 큰 폭으로 절감하였습니다. 추가로, 이미지 데이터셋인 CIFAR-10에 대해서는 Stride 어텐션 패턴을 적용한 트랜스포머 모델이 기존 방식보다 더 좋은 추론 성능을 보이면서 학습 반복당 소요되는 시간도 0.54 -&gt; 0.38로 절감하는데 성공하였습니다.

![](/images/flash-attention/image25.png)

## Pixelated Butterfly

논문에서는 butterfly 어텐션 패턴을 하드웨어 친화적으로 설계(Block, Flat)하여 어텐션, MLP 네트워크를 포함한 다양한 신경망에 대한 GPU 효율을 높여 학습 속도를 개선하는 희소 어텐션 방법론을 제안하였습니다. 논문이 실제로 기여한 바는 더 크지만,  직전에 살펴본 희소 어텐션 방법론의 후속 연구 정도로 이해할 수 있을 것 같습니다.

![](/images/flash-attention/image26.png)

# Experiments

논문에서는 Flash Attention을 적용하여 학습 속도, perplexity, 벤치마크 성능 등 다양한 측면에서 개선이 이뤄질 수 있음을 보였습니다.

## Faster Models

논문 발표 시점 기준으로, Flash Attention을 적용한 결과 단일 노드 한경에서 BERT 학습 시간의 기존 SOTA를 약 15% 정도 개선(20.0m -&gt; 17.4m)하였습니다.

![](/images/flash-attention/image27.png)

GPT-2 모델로 실험한 결과에서는 OpenWebtext dataset에 대해 동일한 perplexity를 달성하기까지 소요된 학습 시간을 Huggingface 및 Megatron-LM에 비해 최대 3.5배까지 단축시킬 수 있었습니다.

![](/images/flash-attention/image28.png)

또한, long-range-arena(LRA) 벤치마크에 대해서 실험한 결과, 기본 트랜스포머 모델을 포함한 다양한 어텐션 방법론과 비교하였을 때, 최대 2.8배 빠른 학습 시간 내에서 가장 우수한 벤치마크 성능을 보였습니다.

![](/images/flash-attention/image29.png)

## Better Models with Longer Sequence

저자는 Flash Attention의 메모리 효율 개선 덕분에 더 큰 컨텍스트 길이를 수용할 수 있게 되었다고 말합니다. GPT-2 기준으로는 약 4배 정도의 컨텍스트 길이를 수용할 수 있었는데요. 비교를 위해 1K부터 4K까지 컨텍스트 길이를 늘려가며 학습 소요 시간 및 벤치마크(OpenWebText) 점수를 비교해본 결과, 기존 Megatron-LM 방식에 비해 4배 더 큰 컨텍스트를 수용하면서도 학습 시간은 오히려 더 빠른 결과를 확인할 수 있었습니다. 수용 가능한 컨텍스트 길이 증가가 따른 벤치마크 점수(perplexity)도 기존 18.2에서 17.5로 개선되었습니다.

![](/images/flash-attention/image30.png)
