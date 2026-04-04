---
layout: default
title: LLM Post-Training RL Methods
permalink: /musings/llm-post-training-rl-methods/
---

# LLM Post-Training RL Methods
_AI-generated research note. Verify critical claims with primary sources._

## TL;DR
- Post-training alignment for LLMs splits into online RL methods and preference-loss optimization methods.
- PPO RLHF is foundational but infra-heavy.
- DPO-family methods are simpler and often more stable in practice.
- GRPO/RLVR are strong for reasoning tasks with verifiable rewards.

## Core objectives

### PPO-style RLHF
$$
\max_\theta \; \mathbb{E}_{x\sim\mathcal{D},y\sim\pi_\theta(\cdot|x)}[r_\phi(x,y)]
-\beta\,\mathrm{KL}(\pi_\theta(\cdot|x)\|\pi_{ref}(\cdot|x))
$$
Plain English: maximize reward-model score while controlling policy drift from the reference policy.
Variables:
- $\pi_\theta$: current policy
- $\pi_{ref}$: reference policy
- $r_\phi$: reward model score
- $\beta$: KL regularization strength

### DPO
$$
\mathcal{L}_{DPO}(\theta)= -\log\sigma\Big(\beta[(\log\pi_\theta(y_w|x)-\log\pi_\theta(y_l|x))-(\log\pi_{ref}(y_w|x)-\log\pi_{ref}(y_l|x))]\Big)
$$
Plain English: increase preferred-response likelihood relative to rejected responses, compared against a reference model margin.
Variables:
- $y_w$: preferred response
- $y_l$: rejected response
- $\beta$: margin scaling parameter

### REINFORCE-style online objective
$$
\nabla_\theta J(\theta)=\mathbb{E}_{y\sim\pi_\theta(\cdot|x)}\left[(R(x,y)-b(x))\nabla_\theta\log\pi_\theta(y|x)\right]
$$
Plain English: increase probability of outputs with above-baseline reward and decrease probability of below-baseline outputs.
Variables:
- $R(x,y)$: reward
- $b(x)$: variance-reduction baseline

## Method families (practical view)
- PPO RLHF: strongest online control, highest stack complexity.
- DPO/ORPO/KTO/RRHF: direct preference optimization, usually easier to run.
- RLOO/REINFORCE-style online: middle-ground online adaptation.
- GRPO + RLVR: effective where reward can be checked automatically (tests, verifiers, symbolic checks).

## Code anchors (major repos)
- OpenAI `lm-human-preferences` — PPO RLHF training loop
- Hugging Face `trl/scripts/dpo.py` — DPO
- Hugging Face `examples/scripts/orpo.py` — ORPO
- Hugging Face `examples/scripts/kto.py` — KTO
- GanjinZero `RRHF/train.py` — RRHF ranking loss
- Hugging Face `trl/scripts/rloo.py` — online RLOO
- Hugging Face `trl/scripts/grpo.py` — GRPO

## References
1. InstructGPT / RLHF: https://arxiv.org/abs/2203.02155
2. DPO: https://arxiv.org/abs/2305.18290
3. ORPO: https://arxiv.org/abs/2403.07691
4. KTO: https://arxiv.org/abs/2402.01306
5. RRHF: https://arxiv.org/abs/2304.05302
6. Hugging Face TRL: https://github.com/huggingface/trl
7. OpenAI lm-human-preferences: https://github.com/openai/lm-human-preferences
8. OpenRLHF: https://github.com/OpenRLHF/OpenRLHF
