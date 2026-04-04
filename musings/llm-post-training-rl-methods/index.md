---
layout: default
title: LLM Post-Training RL Methods
permalink: /musings/llm-post-training-rl-methods/
---

# LLM Post-Training RL Methods
_AI-generated research note. Verify critical claims with primary sources._

## TL;DR
- Post-training alignment for LLMs now splits into two practical families: online RL (PPO/REINFORCE/RLOO/GRPO) and preference-loss optimization (DPO/ORPO/KTO/RRHF).
- PPO RLHF is historically foundational but operationally heavy.
- DPO-family methods are simpler to run and often more stable.
- GRPO + RLVR are particularly useful for reasoning tasks where rewards are verifiable (math/code/tests).

## Why these methods emerged
SFT alone made models more usable, but not reliably preference-aligned for difficult behavior trade-offs.
RLHF introduced reward-driven optimization, then later methods reduced stack complexity:
- PPO RLHF: strong online control, but expensive and tuning-sensitive.
- DPO/ORPO/KTO/RRHF: optimize preference structure more directly, often with lower infra burden.
- Online lightweight variants: preserve on-policy adaptation without full PPO machinery.

## Method cheat sheet

### 1) PPO RLHF
Equation (LaTeX):
```latex
\max_\theta \; \mathbb{E}_{x,y\sim\pi_\theta}[r_\phi(x,y)]
-\beta\,\mathrm{KL}(\pi_\theta(\cdot|x)\|\pi_{ref}(\cdot|x))
```
Plain English: maximize reward-model score while constraining policy drift from reference.
Variables:
- $\theta$: policy params
- $r_\phi$: reward model
- $\beta$: KL strength

Code example source:
- Repo: https://github.com/openai/lm-human-preferences
- File: `lm_human_preferences/train_policy.py`

### 2) DPO
Equation (LaTeX):
```latex
\mathcal{L}_{DPO}(\theta)= -\log\sigma\left(\beta[(\log\pi_\theta(y_w|x)-\log\pi_\theta(y_l|x))-(\log\pi_{ref}(y_w|x)-\log\pi_{ref}(y_l|x))]\right)
```
Plain English: directly increase preferred-vs-rejected margin relative to a reference policy.
Variables:
- $y_w$: preferred response
- $y_l$: rejected response
- $\beta$: margin scaling

Code example source:
- Repo: https://github.com/huggingface/trl
- File: `trl/scripts/dpo.py`

### 3) ORPO
- One-stage objective combining SFT likelihood and preference pressure (odds-ratio style).
- Useful when wanting monolithic training without separate RLHF pipeline.

Code example source:
- Repo: https://github.com/huggingface/trl
- File: `examples/scripts/orpo.py`

### 4) KTO
- Preference optimization using desirability labels inspired by prospect-theoretic framing.
- Helpful when preference annotation style is not strictly pairwise-only.

Code example source:
- Repo: https://github.com/huggingface/trl
- File: `examples/scripts/kto.py`

### 5) RRHF
- Ranking-based objective: encourages model likelihood orderings that match human rankings.

Code example source:
- Repo: https://github.com/GanjinZero/RRHF
- File: `train.py`

### 6) Online lightweight RL (RLOO / REINFORCE-style)
Equation (LaTeX):
```latex
\nabla_\theta J(\theta)=\mathbb{E}_{y\sim\pi_\theta}[ (R-b)\nabla_\theta\log\pi_\theta(y|x)]
```
Plain English: increase probability of above-baseline outputs; decrease below-baseline ones.
Variables:
- $R$: reward
- $b$: baseline

Code example source:
- Repo: https://github.com/huggingface/trl
- File: `trl/scripts/rloo.py`

### 7) GRPO + RLVR
- GRPO normalizes relative outcomes within grouped rollouts to stabilize reasoning updates.
- RLVR uses verifiable rewards (unit tests, symbolic checks, math verifiers), reducing reward-model ambiguity.

Code example source:
- Repo: https://github.com/huggingface/trl
- File: `trl/scripts/grpo.py`

## Practical selection guide
- If you already have high-quality pairwise data and need a simpler stack: start with DPO-family.
- If you need strong online control and can afford infrastructure: PPO-style RLHF.
- If rewards are objectively verifiable (code/math): GRPO/RLVR-style pipelines are often strong choices.

## References
1. InstructGPT / RLHF: https://arxiv.org/abs/2203.02155
2. DPO: https://arxiv.org/abs/2305.18290
3. ORPO: https://arxiv.org/abs/2403.07691
4. KTO: https://arxiv.org/abs/2402.01306
5. RRHF: https://arxiv.org/abs/2304.05302
6. Hugging Face TRL: https://github.com/huggingface/trl
7. OpenAI lm-human-preferences: https://github.com/openai/lm-human-preferences
8. OpenRLHF: https://github.com/OpenRLHF/OpenRLHF
