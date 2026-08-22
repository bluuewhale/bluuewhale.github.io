+++
title = 'SIMA: A Generalist AI Agent for 3D Virtual Environments'
date = '2025-11-16T09:20:13+09:00'
draft = false
translationKey = 'sima-generalist-ai-agent-3d-environments'
slug = 'sima-generalist-ai-agent-3d-environments'
description = '게임 엔진 API 대신 화면과 키보드/마우스만으로 여러 3D 게임을 플레이하는 딥마인드의 범용 에이전트 SIMA를, Behavior Cloning과 Classifier-Free Guidance를 중심으로 정리합니다.'
tags = ['AI', 'Agent', 'Reinforcement Learning', 'DeepMind']
categories = ['AI', 'Agent']

[cover]
image = 'images/sima-generalist-ai-agent-3d-environments/image1.png'
hiddenInSingle = true
+++

딥마인드가 2024년 발표한 [SIMA(Scalable Instructable Multiworld Agent)](https://deepmind.google/blog/sima-generalist-ai-agent-for-3d-virtual-environments/)는 3D 가상 환경 전용 범용 에이전트입니다. 화면 입력과 간단한 자연어 지시만 주어지면, 사람과 거의 같은 방식으로 3D 게임을 플레이합니다.

## 하나의 게임이 아니라 여러 게임을

SIMA가 흥미로운 지점은 특정 게임 하나에 특화된 봇이 아니라는 것입니다. 8개의 게임 스튜디오와 협업해 외계 행성을 탐사하는 No Man's Sky, 자동화 공장을 짓는 Satisfactory, 북유럽 신화 기반 생존 크래프팅 게임 Valheim 등 9개 게임을 대상으로 학습했고, 처음 보는 게임에서도 사람의 지시를 이해하고 플레이할 수 있습니다.

## 사람처럼 플레이한다는 것

SIMA의 가장 두드러진 특징은 입출력 자체가 사람과 동일하다는 점입니다. 입력으로는 게임 화면(픽셀)과 사용자의 음성 또는 텍스트 지시를 받고, 출력으로는 키보드/마우스 조작을 내보냅니다. 즉, 일반적인 게임 봇처럼 엔진의 내부 API를 직접 호출하는 게 아니라, 사람이 앉아서 화면을 보고 손으로 조작하는 것과 똑같은 인터페이스로 게임을 플레이합니다. 이런 특성 때문에 연구진은 SIMA를 "다른 환경에도 빠르게 적용 가능한" 에이전트, 즉 Versatile Agent로 부릅니다.

학습에 사용한 명령들은 "지도를 열어라", "사다리를 올라가라"처럼 10초 이내에 끝낼 수 있는 비교적 짧은 과제들이었습니다.

## Behavior Cloning으로 학습하기

SIMA는 전문가의 행동 데이터를 모아 그 행동을 그대로 따라 하도록 정책을 학습하는 Behavior Cloning 방법론을 채택했습니다.

![](/images/sima-generalist-ai-agent-3d-environments/image1.png)

학습 데이터는 먼저 전문 플레이어의 게임 플레이를 수집한 뒤, 그 플레이 로그(trajectory)에 사후적으로 자연어 명령어를 입히는 방식으로 만들어집니다. 에이전트에게는 게임 플레이 영상과 자연어 명령이 함께 주어지는데, 영상은 사전 학습된 이미지/비디오 인코딩 모델로, 명령은 텍스트 모델로 각각 인코딩됩니다.

모델 자체는 텍스트·이미지·비디오를 함께 처리할 수 있는 cross-attended transformer를 새로 학습시킨 것으로, 긴 시퀀스를 다루기 위해 장기 메모리를 가진 Transformer-XL 구조를 사용합니다. 이 모델은 다음에 취해야 할 8개의 액션(키보드/마우스 조작) 묶음을 출력합니다.

학습 과정에서는 모델이 예측한 액션과 실제 전문가가 취한 액션의 차이를 cross-entropy loss로 계산해 최소화합니다. 여기에 더해, 자연어 명령에 더 높은 가중치를 주기 위해 이미지 생성 모델에서 흔히 쓰이는 Classifier-Free Guidance(CFG) 기법을 차용합니다.

```
𝜋_CFG = 𝜋(image, language) + 𝜆 · (𝜋(image, language) − 𝜋(image))
```

𝜆 값이 커질수록 자연어 명령이 행동 예측에 미치는 영향력도 함께 커집니다.

## 평가 결과

자연어 지시를 내린 뒤 SIMA가 이를 얼마나 잘 수행하는지 측정한 결과, 전체적으로는 50~65% 수준의 성공률을 보였습니다.

![](/images/sima-generalist-ai-agent-3d-environments/image2.png)

명령 종류에 따른 편차는 꽤 뚜렷했습니다. stop, move, drive처럼 이동과 관련된 명령은 상대적으로 잘 수행했지만, cook, build, collect처럼 게임 시스템에 깊이 의존하는 명령은 성공률이 눈에 띄게 떨어졌습니다.

![](/images/sima-generalist-ai-agent-3d-environments/image3.png)

No Man's Sky를 대상으로 전문 플레이어와 직접 비교했을 때는 전문 플레이어가 60%, SIMA가 34%의 성공률을 기록했습니다. 아직 사람 수준에는 미치지 못하지만, 주목할 부분은 따로 있습니다. 해당 게임을 SIMA가 처음 플레이하는 zero-shot 상황에서도 준수한 성능을 보였다는 점인데, 이는 SIMA가 특정 게임의 규칙을 암기한 게 아니라 게임 플레이 자체에 대한 범용적인 지식을 학습했다는 증거로 읽을 수 있습니다.

![](/images/sima-generalist-ai-agent-3d-environments/image4.png)

마지막으로, 자연어 명령과 CFG 적용 여부가 성능에 미치는 영향을 살펴본 결과도 인상적입니다. 자연어 명령이나 CFG를 빼면 성능이 크게 떨어지는데, 이는 SIMA의 행동이 단순히 화면 상태만으로 결정되는 게 아니라 실제로 언어 지시를 따라가고 있다는 것을 보여줍니다.

![](/images/sima-generalist-ai-agent-3d-environments/image5.png)

## References
- [A generalist AI agent for 3D virtual environments (DeepMind)](https://deepmind.google/blog/sima-generalist-ai-agent-for-3d-virtual-environments/)
- [SIMA: A generalist AI agent for 3D virtual environments (arXiv)](https://arxiv.org/pdf/2404.10179)
