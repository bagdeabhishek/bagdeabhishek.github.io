---
layout: default
title: LLM Post-Training RL Methods
permalink: /musings/llm-post-training-rl-methods/
---

# LLM Post-Training RL Methods
_AI-generated research note. Verify critical claims with primary sources._
Status: Completed

## TL;DR
- LLM alignment methods now split into: (1) online RL (PPO/REINFORCE/RLOO/GRPO/RLVR) and (2) preference-loss optimization (DPO/ORPO/KTO/RRHF).
- PPO RLHF is foundational but engineering-heavy.
- DPO-family methods simplify the stack by removing explicit reward-model+PPO loops (or parts of it).
- GRPO + RLVR matter for reasoning tasks where reward is verifiable (math/code/tests).

## Background & Context
LLM post-training began with RLHF pipelines: SFT -> reward model -> PPO policy optimization under KL constraints. This worked well but is expensive and sensitive. Newer methods emerged to preserve alignment gains while reducing infrastructure complexity and optimization fragility.

## A) Classical RLHF with PPO (explicit online RL)

### What it is
Pipeline:
1) SFT on demonstrations
2) Reward model training on preference pairs
3) PPO optimization against reward + KL regularization

### Intuition + why this emerged
- SFT improved formatting/helpfulness, but not enough for nuanced preference optimization.
- PPO RLHF emerged to optimize generated behavior directly using preference-derived reward.
- KL regularization was introduced to prevent reward hacking and excessive policy drift.

### Objective
$$
\max_\theta \; \mathbb{E}_{x\sim\mathcal{D},\;y\sim\pi_\theta(\cdot|x)}\left[r_\phi(x,y)\right]
-\beta\,\mathrm{KL}\!\left(\pi_\theta(\cdot|x)\,\|\,\pi_{ref}(\cdot|x)\right)
$$
Plain English: maximize reward-model score while penalizing divergence from a safer reference policy.
Variables:
- $\theta$: policy parameters
- $x$: prompt
- $y$: sampled completion
- $r_\phi(x,y)$: reward model score
- $\pi_{ref}$: reference policy
- $\beta$: KL penalty strength

Token-level shaping:
$$
r_t^{total} = r_t^{score} - \beta\left(\log\pi_\theta(a_t|s_t)-\log\pi_{ref}(a_t|s_t)\right)
$$
Plain English: reward at each token is task signal minus deviation penalty from reference probabilities.
Variables:
- $a_t$: action/token at step $t$
- $s_t$: state/context at step $t$

PPO clipped objective:
$$
L^{PPO}(\theta)=\mathbb{E}_t\left[\min\left(\rho_tA_t,\;\mathrm{clip}(\rho_t,1-\epsilon,1+\epsilon)A_t\right)\right],
\quad
\rho_t=\frac{\pi_\theta(a_t|s_t)}{\pi_{\theta_{old}}(a_t|s_t)}
$$
Plain English: update using advantage-weighted ratios, clipped to prevent unstable jumps.
Variables:
- $A_t$: advantage
- $\rho_t$: new/old policy ratio
- $\epsilon$: clip threshold

### Code example (line-labeled)
Repo: openai/lm-human-preferences  
File: lm_human_preferences/train_policy.py  
Why relevant: canonical PPO RLHF training loop with adaptive KL and clipped policy objective.

```python
L1: kl = logprobs - ref_logprobs                           # policy vs reference log-prob gap
L2: non_score_reward = -self.kl_ctl.value * kl             # KL penalty term
L3: rewards[:, -1] += scores                               # add terminal reward-model score
L4: ratio = tf.exp(logprob - old_logprob)                  # PPO importance ratio
L5: pg_losses = -advantages * ratio                        # unclipped policy-gradient term
L6: pg_losses2 = -advantages * tf.clip_by_value(ratio, 1.0 - cliprange, 1.0 + cliprange)  # clipped term
L7: pg_loss = tf.reduce_mean(tf.maximum(pg_losses, pg_losses2))  # PPO surrogate
```

### Practical notes
- Strong online adaptation and historical validation.
- Expensive stack (policy/ref/reward/rollout infra).
- Sensitive to reward scaling and KL scheduling.

---

## B) DPO-family (RL-free / RL-light preference optimization)

### Intuition + why this family emerged
- If labels are winner-vs-loser, optimize that margin directly.
- This avoids separate reward-model training and often avoids PPO complexity.
- Lower tuning burden and better stability in many practical settings.

### B1) DPO
$$
\mathcal{L}_{DPO}(\theta)= -\log\sigma\Big(\beta[(\log\pi_\theta(y_w|x)-\log\pi_\theta(y_l|x))-(\log\pi_{ref}(y_w|x)-\log\pi_{ref}(y_l|x))]\Big)
$$
Plain English: directly increase preferred-vs-rejected likelihood margin, relative to reference.
Variables:
- $y_w$: preferred response
- $y_l$: rejected response
- $\beta$: margin scaling

Code example:
Repo: huggingface/trl  
File: trl/scripts/dpo.py  

```python
L1: dataset = load_dataset(script_args.dataset_name, name=script_args.dataset_config)  # load preference pairs
L2: trainer = DPOTrainer(model, args=training_args, train_dataset=dataset[train_split], eval_dataset=dataset[test_split])  # DPO setup
L3: trainer.train()  # optimize DPO objective
```

### B2) ORPO
What it is: single-stage objective combining SFT likelihood + odds-ratio style preference pressure.

Code example:
Repo: huggingface/trl  
File: examples/scripts/orpo.py

```python
L1: trainer = ORPOTrainer(model, args=training_args, train_dataset=train_ds, eval_dataset=eval_ds, processing_class=tokenizer)  # monolithic ORPO training
L2: trainer.train()  # run optimization
```

### B3) KTO
What it is: preference optimization with desirability-style labels and prospect-theory motivation.

Code example:
Repo: huggingface/trl  
File: examples/scripts/kto.py

```python
L1: model = AutoModelForCausalLM.from_pretrained(model_name)      # policy model
L2: ref_model = AutoModelForCausalLM.from_pretrained(model_name)  # frozen reference
L3: trainer = KTOTrainer(model, ref_model, args=training_args, train_dataset=train_ds, processing_class=tokenizer)  # KTO setup
L4: trainer.train()  # optimize KTO objective
```

### B4) RRHF
What it is: ranking-based objective aligning likelihood ordering with human ranking order.

Code example:
Repo: GanjinZero/RRHF  
File: train.py

```python
L1: diff = scores.unsqueeze(0) - scores.unsqueeze(-1)             # model score pairwise diff
L2: rw_diff = rw_scores.unsqueeze(0) - rw_scores.unsqueeze(-1)    # human ranking pairwise diff
L3: aval = torch.bitwise_and(rw_diff > 0, diff < 0)[0]            # where ordering is wrong
L4: rrhf_loss = -diff[aval].sum()                                 # penalize rank violations
L5: loss = rrhf_weight * rrhf_loss + sft_loss                     # anchor with SFT
```

### Practical notes
- DPO-family often wins on simplicity and reproducibility.
- Can underperform if preference data quality is weak/noisy.
- Reference model choice and beta scaling still matter.

---

## C) Online preference optimization variants (middle ground)

### Intuition + why this emerged
- Pure offline training can become stale as policy changes.
- Online sampling keeps updates aligned with current policy behavior.
- REINFORCE-style variants reduce PPO machinery while preserving on-policy updates.

### Objective
$$
\nabla_\theta J(\theta)=\mathbb{E}_{y\sim\pi_\theta(\cdot|x)}\left[(R(x,y)-b(x))\,\nabla_\theta\log\pi_\theta(y|x)\right]
$$
Plain English: increase probabilities of above-baseline outputs, reduce below-baseline outputs.
Variables:
- $R(x,y)$: reward
- $b(x)$: baseline for variance reduction

### Code example
Repo: huggingface/trl  
File: trl/scripts/rloo.py

```python
L1: reward_funcs = []                                              # define online reward sources
L2: if script_args.reward_model_name_or_path: reward_funcs.append(script_args.reward_model_name_or_path)  # add RM
L3: trainer = RLOOTrainer(model=model, reward_funcs=reward_funcs, args=training_args, train_dataset=train_ds)  # online trainer
L4: trainer.train()                                                # on-policy updates
```

### Practical notes
- More adaptive than fully offline objectives.
- Still needs rollout infra and robust reward design.

---

## D) GRPO + RLVR for reasoning

### Intuition + why this emerged
- Reasoning tasks (math/code) benefit from grouped comparisons and verifiable correctness.
- GRPO stabilizes optimization with relative/group normalization.
- RLVR uses deterministic verifiers (tests, checkers) to reduce reward ambiguity.

### Code example
Repo: huggingface/trl  
File: trl/scripts/grpo.py

```python
L1: trainer = GRPOTrainer(model=model, args=training_args, train_dataset=train_ds, eval_dataset=eval_ds, reward_funcs=reward_funcs)  # grouped RL trainer
L2: trainer.train()  # optimize with group-relative rewards
```

### Practical notes
- Strong fit for tasks with objective correctness checks.
- Less ideal when reward is subjective/hard to verify.

---

## Selection guide
- Need highest online control and can pay infra cost: PPO RLHF.
- Need simpler/easier operational pipeline with pairwise data: DPO-family.
- Need online adaptation but lighter than PPO: RLOO/REINFORCE-style.
- Need reasoning gains with checkable outcomes: GRPO + RLVR.

## References
1. InstructGPT / RLHF: https://arxiv.org/abs/2203.02155
2. DPO: https://arxiv.org/abs/2305.18290
3. ORPO: https://arxiv.org/abs/2403.07691
4. KTO: https://arxiv.org/abs/2402.01306
5. RRHF: https://arxiv.org/abs/2304.05302
6. Hugging Face TRL: https://github.com/huggingface/trl
7. OpenAI lm-human-preferences: https://github.com/openai/lm-human-preferences
8. OpenRLHF: https://github.com/OpenRLHF/OpenRLHF
