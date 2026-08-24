+++
title = 'SIMA: A Generalist AI Agent for 3D Virtual Environments'
date = '2025-11-16T09:20:13+09:00'
draft = false
translationKey = 'sima-generalist-ai-agent-3d-environments'
slug = 'sima-generalist-ai-agent-3d-environments-en'
aliases = ['/posts/sima-generalist-ai-agent-3d-environments-en/']
description = "How DeepMind's SIMA plays multiple 3D games through nothing but screen pixels and keyboard/mouse output, trained with behavior cloning and classifier-free guidance."
tags = ['AI', 'Agent', 'Reinforcement Learning', 'DeepMind']
categories = ['AI', 'Agent']

[cover]
image = 'images/sima-generalist-ai-agent-3d-environments/image1.png'
hiddenInSingle = true
+++

[SIMA (Scalable Instructable Multiworld Agent)](https://deepmind.google/blog/sima-generalist-ai-agent-for-3d-virtual-environments/), which DeepMind published in 2024, is a generalist agent built for 3D virtual environments. Give it a screen and a simple natural-language instruction, and it plays a 3D game almost the way a human would.

## Not One Game, But Many

SIMA isn't a bot tuned for one specific game, and that's what makes it interesting. Working with eight game studios, DeepMind trained it across nine titles, including No Man's Sky (exploring alien planets), Satisfactory (building automated factories on an alien world), and Valheim (a Norse-mythology survival crafting game). It can follow human instructions and play games it has never seen before.

## Playing Like a Human

SIMA's most distinctive trait is that its inputs and outputs match a human's exactly. It takes in the game screen (raw pixels) plus a spoken or typed instruction, and it outputs keyboard and mouse actions. Unlike a typical game bot that calls into the engine's internal API, SIMA plays through the same interface a person uses: watching a screen and operating a keyboard and mouse. That's why the researchers call it a versatile agent, one that adapts quickly to environments it hasn't seen.

The instructions used during training were relatively short, completable in under 10 seconds: things like "open the map" or "climb the ladder."

## Training with Behavior Cloning

SIMA learns through behavior cloning: collect data on expert behavior, then train a policy to reproduce it directly.

![](/images/sima-generalist-ai-agent-3d-environments/image1.png)

The training data starts with recordings of expert players, and natural-language commands get attached to those trajectories after the fact. The agent receives both a gameplay video and a language instruction; the video is encoded with a pretrained image/video encoder, and the instruction with a text model.

The model itself is a newly trained cross-attended transformer capable of handling text, image, and video together, built on Transformer-XL for its long-context memory. It outputs a set of 8 actions (keyboard/mouse operations) to take next.

Training minimizes the cross-entropy loss between the model's predicted actions and what the expert did. On top of that, SIMA borrows classifier-free guidance (CFG), a technique common in image generation, to give natural-language instructions more weight:

```
𝜋_CFG = 𝜋(image, language) + 𝜆 · (𝜋(image, language) − 𝜋(image))
```

The larger 𝜆 gets, the more the language instruction influences the predicted action.

## Evaluation

Measured on how well SIMA follows a natural-language instruction once it's given, success rates land in the 50-65% range overall.

![](/images/sima-generalist-ai-agent-3d-environments/image2.png)

The gap between command types is fairly stark. SIMA handles movement-related commands like stop, move, and drive well, but success drops noticeably on commands that lean heavily on game systems, like cook, build, and collect.

![](/images/sima-generalist-ai-agent-3d-environments/image3.png)

Compared directly against expert human players on No Man's Sky, SIMA hit a 34% success rate versus the experts' 60%. It's not at human level yet, but SIMA performed reasonably well even in zero-shot conditions, playing that game for the first time. That reads less like memorized game-specific rules and more like evidence SIMA has learned something generalizable about how to play games at all.

![](/images/sima-generalist-ai-agent-3d-environments/image4.png)

Finally, dropping natural-language instructions or CFG causes performance to fall off sharply, which shows SIMA follows the language instruction rather than reacting to screen state alone.

![](/images/sima-generalist-ai-agent-3d-environments/image5.png)

## References
- [A generalist AI agent for 3D virtual environments (DeepMind)](https://deepmind.google/blog/sima-generalist-ai-agent-for-3d-virtual-environments/)
- [SIMA: A generalist AI agent for 3D virtual environments (arXiv)](https://arxiv.org/pdf/2404.10179)
