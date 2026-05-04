---
layout: default
title: LLM Post-Training RL Methods
permalink: /musings/llm-post-training-rl-methods/
---

# LLM Post-Training RL Methods
_AI-assisted research synthesis. Verify critical claims with primary sources._
Status: Completed
Last updated: 2026-04-15T21:04:55+05:30
Mode: comparative survey

## Summary
- RLHF (PPO-based) remains the reference point for aligning LLMs post-training, but newer methods DPO, SimPO, KTO, and GRPO address different weaknesses. None is categorically better.
- DPO and SimPO remove the reward model, simplifying training and avoiding reward hacking. SimPO uses length-normalized reference-free rewards.
- KTO works with binary feedback (thumbs up/down) instead of pairwise preferences, lowering data requirements.
- GRPO enables group-relative reward signals without a value function, important for reasoning-centric training like DeepSeek-R1.

The practical takeaway: choose the method that matches your data format and compute budget, not the one with the best headline number.

## Overview
Post-training RL methods take a base LLM (after pretraining and SFT) and further align it using human or AI feedback. RLHF with PPO has been the dominant approach since InstructGPT (2022). However, PPO's complexity (reward model, value function, reference model, four models in memory) has driven a wave of simpler alternatives.

This note compares DPO, SimPO, KTO, and GRPO — the most discussed alternatives — on mechanism, data requirements, computational cost, and when to use each.

## Background
RLHF was introduced by Christiano et al. (2017) and scaled to language models by InstructGPT (Ouyang et al., 2022). The standard pipeline:

1. Collect human preference data (pairs of model outputs, human picks the better one)
2. Train a reward model on those preferences
3. Fine-tune the policy (LLM) with PPO against the reward model, with a KL penalty toward a reference model

Problems with PPO: four models in memory (policy, reference, reward, value), training instability, reward hacking, and the complexity of maintaining a separate reward model.

## Core Analysis

### DPO (Direct Preference Optimization)
**Mechanism:** Eliminates the reward model entirely. The policy itself serves as an implicit reward model. The loss function reparameterizes the Bradley-Terry preference model directly in terms of the policy and reference model.

**Key equation:**

$$
\mathcal{L}_{\text{DPO}}(\pi_\theta; \pi_{\text{ref}}) = -\mathbb{E}_{(x,y_w,y_l)\sim\mathcal{D}}\left[\log\sigma\left(\beta\log\frac{\pi_\theta(y_w|x)}{\pi_{\text{ref}}(y_w|x)} - \beta\log\frac{\pi_\theta(y_l|x)}{\pi_{\text{ref}}(y_l|x)}\right)\right]
$$

Plain English: DPO increases the log-ratio of the policy over the reference for the preferred output (y_w) relative to the dispreferred output (y_l). The hyperparameter β controls how far the policy can move from the reference. No explicit reward model is trained.

**Data:** Same pairwise preferences as RLHF (preferred vs. dispreferred).

**Compute:** Single-stage training. Two models in memory (policy + reference). No reward model, no value function, no PPO loop. Faster and simpler than RLHF.

**When to use:** When you have pairwise preference data and want a simpler, more stable training pipeline than PPO. Strong results on standard alignment benchmarks.

### SimPO (Simple Preference Optimization)
**Mechanism:** Reference-free variant of DPO. Instead of comparing policy to a frozen reference model, SimPO compares the average log-probability of the sequence (length-normalized). Includes a target reward margin (γ) between preferred and dispreferred outputs.

**Data:** Same pairwise preferences as DPO/RLHF.

**Compute:** Single model in memory (no reference model). Length normalization is the key change from DPO.

**When to use:** When you want the simpler setup of DPO but with better length control. SimPO reduces verbosity bias by normalizing by sequence length. Often outperforms DPO on benchmarks with less verbosity inflation.

### KTO (Kahneman-Tversky Optimization)
**Mechanism:** Works with binary feedback (good/bad, thumbs up/down) instead of pairwise comparisons. Based on prospect theory: humans evaluate outcomes relative to a reference point, and losses loom larger than gains.

**Data:** Binary labels per output ("good" or "bad"), not paired comparisons. Can also use only positive examples if negatives are unavailable.

**Compute:** Similar to DPO in complexity (two models: policy + reference, no reward model).

**When to use:** When you have binary feedback data (thumbs up/down, star ratings) rather than pairwise preferences. Particularly useful when collecting pairwise comparisons is expensive or when you have existing binary-labeled data.

### GRPO (Group Relative Policy Optimization)
**Mechanism:** Designed for tasks where a reward signal can be computed automatically (verifiable tasks like math, code). Samples a group of outputs for the same prompt, computes a reward for each (e.g., correctness), normalizes within the group, and optimizes the policy using the group-relative advantage. No value function needed.

**Data:** Prompts with verifiable rewards (math answers, code execution results). No human preference data needed.

**Compute:** Policy model only (no reference model, no value function, no reward model — reward comes from a verifier function). Used in DeepSeek-R1 for reasoning training.

**When to use:** For reasoning-centric training where ground-truth rewards are available (math, code, formal reasoning). GRPO is the key innovation in DeepSeek-R1 that enables RL-driven reasoning improvement.

### Why the tradeoffs exist

The core tension is between:
- **Data efficiency**: how much and what kind of human feedback is needed
- **Training complexity**: how many models, how much memory, how many stages
- **Reward robustness**: how resistant the method is to reward hacking

RLHF (PPO) has the highest data efficiency (learns from pairwise preferences) but the highest complexity (four models). DPO/SimPO reduce complexity at the cost of potentially less fine-grained control. KTO reduces data requirements further but may miss nuance in preference strength. GRPO is orthogonal — it optimizes for a different data regime (verifiable rewards).

### Limits / common misunderstandings

A common misunderstanding is that DPO/SimPO "replaces" RLHF. They replace the PPO training stage but still require the same preference data. The difference is algorithmic, not in data requirements.

Another misunderstanding is that reference-free methods (SimPO) eliminate the need for a frozen model. They still implicitly regularize toward the pre-trained distribution through length normalization and the policy update itself.

A third misunderstanding is that GRPO is only for math. It is applicable to any domain with a verifiable reward signal, including code (passes tests) and formal reasoning (proof checker).

## Evidence and Sources

### Claim cluster 1: DPO matches PPO quality with simpler training
- The original DPO paper (Rafailov et al., 2023) showed DPO matching or exceeding PPO on controlled sentiment and summarization tasks.
- Comparison studies find DPO competitive on standard alignment benchmarks (AlpacaEval, MT-Bench) when properly tuned.
- The simplification is real: DPO requires 2-3x less GPU memory and fewer training stages.

### Claim cluster 2: SimPO improves on DPO's verbosity bias
- The SimPO paper (Meng et al., 2024) demonstrates that length normalization reduces the tendency of DPO to prefer longer outputs.
- On AlpacaEval 2.0, SimPO outperforms DPO while producing shorter responses.

### Claim cluster 3: KTO works with binary feedback
- The KTO paper (Ethayarajh et al., 2024) shows that binary-labeled data can produce alignment quality approaching pairwise preference methods.
- This lowers the cost of data collection significantly.

### Claim cluster 4: GRPO is effective for reasoning
- DeepSeek-R1 (DeepSeek-AI, 2025) used GRPO as the core RL method for training reasoning capabilities.
- The key insight: group-relative normalization provides a stable training signal without a value function, making it well-suited to reasoning tasks with sparse rewards.

## Uncertainties and Competing Views

High-confidence claims:
- DPO matches PPO for standard alignment tasks at lower complexity.
- GRPO works for reasoning training with verifiable rewards.
- Binary feedback (KTO) is sufficient for many alignment tasks.

Medium-confidence claims:
- SimPO consistently outperforms DPO (more comparative studies needed across diverse tasks).
- KTO matches pairwise methods at scale (binary feedback may lose nuance at the tail).

Competing views:
- Some researchers argue PPO remains superior for fine-grained preference optimization, especially when reward modeling can be improved iteratively.
- Others argue that the choice of method matters less than data quality and scale.

What evidence would change the conclusion:
- Large-scale head-to-head comparisons across all methods on a unified benchmark.
- Production A/B tests showing real user preference differences between methods.

## Practical Takeaways

### Decision framework

| Your situation | Recommended method |
|----------------|-------------------|
| Pairwise preferences, standard alignment | DPO |
| DPO but output too verbose | SimPO |
| Only binary feedback available | KTO |
| Verifiable task with ground-truth reward | GRPO |
| Maximum quality, can afford complexity | RLHF (PPO) |
| Quick experiment, minimal setup | SimPO (simplest) |

### Implementation notes

- All methods are available in TRL (HuggingFace), Axolotl, and Unsloth.
- Start with your data: the data format determines which method you can use.
- β (the KL penalty coefficient) is the most important hyperparameter across all methods.
- For GRPO, the group size and reward normalization scheme need tuning.

## References
1. [DPO — Rafailov et al. (2023), Direct Preference Optimization](https://arxiv.org/abs/2305.18290) — Primary. Original DPO paper.
2. [SimPO — Meng et al. (2024), SimPO: Simple Preference Optimization](https://arxiv.org/abs/2405.14734) — Primary. Reference-free variant with length normalization.
3. [KTO — Ethayarajh et al. (2024), KTO: Model Alignment as Prospect Theoretic Optimization](https://arxiv.org/abs/2402.01306) — Primary. Binary feedback optimization.
4. [DeepSeek-R1 — DeepSeek-AI (2025), DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via RL](https://arxiv.org/abs/2501.12948) — Primary. Introduces GRPO for reasoning training.
5. [TRL — Transformer Reinforcement Learning](https://github.com/huggingface/trl) — Primary. Standard library for RLHF/DPO/GRPO implementations.
6. [InstructGPT — Ouyang et al. (2022), Training Language Models to Follow Instructions](https://arxiv.org/abs/2203.02155) — Primary. Original RLHF paper for LLMs.
