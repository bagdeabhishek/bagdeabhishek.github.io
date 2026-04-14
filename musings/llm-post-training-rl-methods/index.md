---
layout: default
title: LLM Post-Training RL Methods
permalink: /musings/llm-post-training-rl-methods/
---

# LLM Post-Training RL Methods
_AI-generated research draft. Verify critical claims with primary sources._
Status: Completed
Last updated: 2026-04-04T19:36:21+05:30

## TL;DR
- LLM alignment methods now split into: (1) online RL (PPO/REINFORCE/GRPO/RLVR) and (2) preference-loss optimization (DPO/ORPO/KTO/RRHF).
- PPO-based RLHF is foundational, but engineering-heavy; many teams prefer simpler DPO-family training.
- GRPO + RLVR became important for reasoning (math/code) where rewards are verifiable.
- There is no universal winner: method choice depends on data type (pairwise preference vs verifiable reward), compute budget, and deployment goals.

## Background & Context
LLM post-training started with RLHF pipelines that train a reward model from human preferences and then optimize the policy with PPO under KL constraints. This works but is expensive and fragile. Newer methods aim to keep alignment quality while reducing complexity.

## Taxonomy + Math + Algorithms (integrated)

### A) Classical RLHF with PPO (explicit online RL)
Pipeline:
1) SFT on demonstrations
2) Reward model on preference pairs
3) PPO optimization against reward + KL regularization

Core objective:
Equation (LaTeX):
```latex
\max_\theta \; \mathbb{E}_{x\sim\mathcal{D},\;y\sim\pi_\theta(\cdot|x)}\left[r_\phi(x,y)\right]
-\beta\,\mathrm{KL}\!\left(\pi_\theta(\cdot|x)\,\|\,\pi_{ref}(\cdot|x)\right)
```
Plain English: Train the model to get higher reward-model scores, while penalizing it if it drifts too far from a reference policy.
Variables:
- $\theta$: current policy model parameters
- $x$: prompt sampled from dataset $\mathcal{D}$
- $y$: completion sampled from current policy
- $\pi_\theta$: current policy distribution
- $\pi_{ref}$: reference (usually SFT/base) policy
- $r_\phi(x,y)$: reward model score
- $\beta$: KL penalty strength
- $\mathrm{KL}(\cdot\|\cdot)$: divergence between policies

Token-level KL shaping (common in practice):
Equation (LaTeX):
```latex
r_t^{total} = r_t^{score} - \beta\left(\log\pi_\theta(a_t|s_t)-\log\pi_{ref}(a_t|s_t)\right)
```
Plain English: At each token step, total reward is task/reward-model signal minus a penalty for deviating from the reference model token probability.
Variables:
- $t$: token timestep
- $r_t^{score}$: base reward contribution at step $t$
- $r_t^{total}$: KL-shaped reward at step $t$
- $a_t$: chosen token/action at step $t$
- $s_t$: state/context at step $t$
- $\beta$: KL coefficient

PPO clipped surrogate:
Equation (LaTeX):
```latex
L^{PPO}(\theta)=\mathbb{E}_t\left[\min\left(\rho_tA_t,\;\mathrm{clip}(\rho_t,1-\epsilon,1+\epsilon)A_t\right)\right],
\quad
\rho_t=\frac{\pi_\theta(a_t|s_t)}{\pi_{\theta_{old}}(a_t|s_t)}
```
Plain English: Update the policy using advantage-weighted importance ratios, but clip updates so each step cannot change policy probability too aggressively.
Variables:
- $L^{PPO}(\theta)$: PPO objective
- $\rho_t$: probability ratio (new policy vs old policy)
- $A_t$: advantage estimate at step $t$
- $\epsilon$: clipping threshold
- $\theta_{old}$: parameters before current PPO update

Why this matters:
- strong online adaptation
- historically validated (InstructGPT)

Main pain points:
- reward/KL tuning sensitivity
- expensive multi-model training stack

Intuition + why this method emerged:
- Early instruction tuning alone (SFT) improved formatting/helpfulness but was weak at nuanced preference optimization.
- RLHF+PPO emerged as a way to optimize long-horizon generation quality directly against a learned reward signal, while KL regularization prevented the policy from drifting into reward hacking.

Code example (major repo): OpenAI `lm-human-preferences` (PPO RLHF)
```python
# Repo: https://github.com/openai/lm-human-preferences
# File: lm_human_preferences/train_policy.py

# L1: KL-shaped reward (reward score + KL control)
kl = logprobs - ref_logprobs                           # compare policy vs reference
non_score_reward = -self.kl_ctl.value * kl             # adaptive KL penalty
rewards[:, -1] += scores                               # add task/reward-model score on terminal token

# L2: PPO policy-ratio objective with clipping
ratio = tf.exp(logprob - old_logprob)                  # importance ratio
pg_losses = -advantages * ratio                        # unclipped PG objective
pg_losses2 = -advantages * tf.clip_by_value(ratio, 1.0 - self.hparams.ppo.cliprange, 1.0 + self.hparams.ppo.cliprange)
pg_loss = tf.reduce_mean(tf.maximum(pg_losses, pg_losses2))

# L3: trainer step loop
rollouts = self.policy.respond(queries, length=self.hparams.task.response_length)
train_stats = self.train(rollouts=rollouts)            # multiple minibatch PPO epochs
self.kl_ctl.update(stats['objective/kl'], self.hparams.ppo.batch_size)  # adaptive KL controller
```

### B) DPO-family (RL-free / RL-light preference optimization)

#### DPO
For each pair $(x,y_w,y_l)$:
Equation (LaTeX):
```latex
\mathcal{L}_{DPO}(\theta)= -\log\sigma\Big(\beta[(\log\pi_\theta(y_w|x)-\log\pi_\theta(y_l|x))-(\log\pi_{ref}(y_w|x)-\log\pi_{ref}(y_l|x))]\Big)
```
Plain English: Make preferred responses more likely than rejected responses, relative to a reference model margin, using a logistic loss.
Variables:
- $\mathcal{L}_{DPO}(\theta)$: DPO loss
- $x$: prompt
- $y_w$: preferred (“winner”) response
- $y_l$: rejected (“loser”) response
- $\pi_\theta$: trainable policy
- $\pi_{ref}$: frozen reference policy
- $\beta$: inverse-temperature / margin scaling term
- $\sigma(\cdot)$: sigmoid function

Interpretation: directly increases preferred-vs-rejected log-prob margin relative to reference model; avoids explicit reward model + PPO loop.

#### ORPO
Monolithic training: NLL/SFT term plus odds-ratio preference penalty in one stage (reference-model-free formulation).

#### KTO
Uses binary desirability labels and prospect-theoretic human-aware loss design, rather than pairwise-only preference likelihood.

#### RRHF
Aligns model likelihood ordering to human ranking via ranking loss over sampled responses.

Why this family took off:
- simpler and often more stable than PPO stacks
- lower infra and hyperparameter burden

Intuition + why this method family emerged:
- Researchers observed that if preferences are pairwise (winner vs loser), you can optimize the policy margin directly without fitting a separate reward model and running PPO.
- This cuts failure modes from reward overfitting and PPO instability while preserving much of the alignment gain.

Code examples (major repos):

DPO — Hugging Face TRL (`trl/scripts/dpo.py`)
```python
# Repo: https://github.com/huggingface/trl
# File: trl/scripts/dpo.py

# L1: load preference dataset
dataset = load_dataset(script_args.dataset_name, name=script_args.dataset_config)

# L2: initialize trainer with DPO objective
trainer = DPOTrainer(
    model,
    args=training_args,
    train_dataset=dataset[script_args.dataset_train_split],
    eval_dataset=dataset[script_args.dataset_test_split] if training_args.eval_strategy != "no" else None,
    peft_config=peft_config,
)

# L3: optimize
trainer.train()
```

ORPO — Hugging Face TRL experimental ORPO (`examples/scripts/orpo.py`)
```python
# Repo: https://github.com/huggingface/trl
# File: examples/scripts/orpo.py

# L1: ORPO trainer setup (single-stage SFT + odds-ratio preference pressure)
trainer = ORPOTrainer(
    model,
    args=training_args,
    train_dataset=dataset[script_args.dataset_train_split],
    eval_dataset=dataset[script_args.dataset_test_split] if training_args.eval_strategy != "no" else None,
    processing_class=tokenizer,
    peft_config=get_peft_config(model_args),
)

# L2: train
trainer.train()
```

KTO — Hugging Face TRL experimental KTO (`examples/scripts/kto.py`)
```python
# Repo: https://github.com/huggingface/trl
# File: examples/scripts/kto.py

# L1: policy + reference models for KTO-style preference optimization
model = AutoModelForCausalLM.from_pretrained(model_args.model_name_or_path)
ref_model = AutoModelForCausalLM.from_pretrained(model_args.model_name_or_path)

# L2: initialize KTO trainer
trainer = KTOTrainer(
    model,
    ref_model,
    args=training_args,
    train_dataset=dataset[script_args.dataset_train_split],
    processing_class=tokenizer,
)

# L3: train
trainer.train()
```

RRHF — GanjinZero/RRHF (`train.py`)
```python
# Repo: https://github.com/GanjinZero/RRHF
# File: train.py

# L1: ranking-aware RRHF loss (penalize wrong orderings vs human scores)
def rrhf_loss(self, scores, idxs, rw_scores):
    diff = scores.unsqueeze(0) - scores.unsqueeze(-1)
    rw_diff = rw_scores.unsqueeze(0) - rw_scores.unsqueeze(-1)
    aval = torch.bitwise_and(rw_diff > 0, diff < 0)[0]
    return -diff[aval].sum()

# L2: combine RRHF ranking loss + SFT anchor
loss = self.args.rrhf_weight * rrhf_loss + sft_loss
```

### C) Online preference optimization variants (middle ground)
Goal: retain online sampling advantages with lighter objectives than classical PPO-RLHF.

- Online IPO / IPO-MD formulations
- Online DPO variants for multi-objective or continual settings
- REINFORCE-style simplifications revisiting whether PPO complexity is always needed

REINFORCE-style objective (generic form):
Equation (LaTeX):
```latex
\nabla_\theta J(\theta)=\mathbb{E}_{y\sim\pi_\theta(\cdot|x)}\left[(R(x,y)-b(x))\,\nabla_\theta\log\pi_\theta(y|x)\right]
```
Plain English: Increase probability of outputs with above-baseline reward and decrease probability of below-baseline outputs.
Variables:
- $J(\theta)$: expected return objective
- $R(x,y)$: reward for output $y$ on prompt $x$
- $b(x)$: baseline to reduce gradient variance
- $\nabla_\theta\log\pi_\theta(y|x)$: policy score-function gradient

Typical algorithm pattern:
1) sample fresh outputs from current policy
2) score with preference model/judge
3) update with preference objective online

Tradeoff:
- better adaptability than pure offline training
- still requires online data and robust infra

Intuition + why this method class emerged:
- Offline preference methods are efficient but can get stale as policy distribution shifts.
- Online variants re-sample from the current policy each round and update immediately, keeping gradients matched to current behavior.
- REINFORCE-style simplifications became attractive because they remove some PPO machinery while retaining online policy improvement.

Code example (major repo): Hugging Face TRL RLOO (`trl/scripts/rloo.py`)
```python
# Repo: https://github.com/huggingface/trl
# File: trl/scripts/rloo.py

# L1: define reward functions used online during rollout/update
reward_funcs = []
if script_args.reward_model_name_or_path:
    reward_funcs.append(script_args.reward_model_name_or_path)

# L2: initialize online trainer
trainer = RLOOTrainer(
    model=model_args.model_name_or_path,
    reward_funcs=reward_funcs,
    args=training_args,
    train_dataset=dataset[script_args.dataset_train_split],
    peft_config=get_peft_config(model_args),
)

# L3: online optimization loop (inside trainer)
trainer.train()
```

### D) Constitutional AI / RLAIF
Use constitutional principles and AI critiques/preferences to reduce dependence on human pair labels.

Conceptual flow:
1) self-critique + revision supervised phase
2) preference data generated using AI constitutional judgments
3) RL or preference optimization on this AI feedback signal

Strength:
- scales safety supervision with less direct human labeling

Risk:
- model-judge bias / constitution design bias

Intuition + why this method emerged:
- Human preference labeling is expensive and can bottleneck safety iteration.
- Constitutional AI reframes alignment as: "write principles first, then use AI to critique/revise outputs against those principles," creating scalable synthetic preference signals (RLAIF).

Code example (practical open-source pattern in major repo): RLAIF-style data consumed via DPO in TRL
```python
# Repo: https://github.com/huggingface/trl
# File: trl/scripts/dpo.py
# Note: in practice, `dataset_name` can point to AI-judged/constitutional preference data.

# L1: load preference pairs (human- or AI-judged)
dataset = load_dataset(script_args.dataset_name, name=script_args.dataset_config)

# L2: run preference optimization over that data
trainer = DPOTrainer(
    model,
    args=training_args,
    train_dataset=dataset[script_args.dataset_train_split],
)

# L3: optimize
trainer.train()
```

Practical note:
- Public frontier labs rarely release full constitutional self-critique training pipelines end-to-end; open repos most often expose the downstream optimization stage on constitution-derived preference data.

### E) GRPO + RLVR (verifiable reward RL)

#### RLVR setup
Use externally verifiable reward function:
Equation (LaTeX):
```latex
r(x,y)\in\{0,1\}\;\text{or}\;\mathbb{R}
```
Plain English: Reward comes from objective checkers (like tests or exact answer match), not a learned reward model.
Variables:
- $r(x,y)$: verifier-provided reward
- $x$: prompt/input
- $y$: generated output
- $\{0,1\}$: binary pass/fail reward case
- $\mathbb{R}$: real-valued reward case

#### GRPO-style relative advantage
For a prompt, sample $G$ candidates, rewards $r_i$, then normalize within group:
Equation (LaTeX):
```latex
A_i=\frac{r_i-\mu_r}{\sigma_r+\varepsilon},
\quad
\mu_r=\frac{1}{G}\sum_i r_i,
\quad
\sigma_r^2=\frac{1}{G}\sum_i(r_i-\mu_r)^2
```
Plain English: Each candidate is judged relative to other candidates for the same prompt; higher-than-group-average rewards get positive advantage.
Variables:
- $G$: group size (number of sampled candidates per prompt)
- $r_i$: reward of candidate $i$
- $\mu_r$: group mean reward
- $\sigma_r$: group reward standard deviation
- $\varepsilon$: small stabilizer constant
- $A_i$: normalized relative advantage for candidate $i$

Then apply PPO-like update using these relative advantages.

Why this is effective:
- excellent for math/code/reasoning where correctness is checkable
- reduces reliance on expensive human preference labels

Limit:
- subjective tasks (style/helpfulness nuance) still need preference-oriented supervision

Intuition + why this method emerged:
- Preference labels are noisy/expensive for domains where correctness is objectively checkable.
- RLVR replaces subjective reward modeling with hard verifiers (tests, exact matches, parsers).
- GRPO then stabilizes learning by comparing candidates within each prompt-group, reducing variance from absolute reward scale.

Code example (major repo): Hugging Face TRL GRPO (`trl/scripts/grpo.py`)
```python
# Repo: https://github.com/huggingface/trl
# File: trl/scripts/grpo.py

# L1: map string names to verifier-like reward functions
reward_funcs_registry = {
    "accuracy_reward": accuracy_reward,
    "reasoning_accuracy_reward": reasoning_accuracy_reward,
    "think_format_reward": think_format_reward,
}

# L2: initialize GRPO with chosen reward functions
trainer = GRPOTrainer(
    model=model_args.model_name_or_path,
    reward_funcs=reward_funcs,
    args=training_args,
    train_dataset=dataset[script_args.dataset_train_split],
    peft_config=get_peft_config(model_args),
)

# L3: optimize group-relative objective
trainer.train()
```

## Comparative Snapshot

| Method family | Signal type | Online sampling | Extra reward model | Strength | Weakness |
|---|---|---:|---:|---|---|
| PPO-RLHF | Human pairwise via RM | Yes | Yes | strong control | complex, brittle |
| DPO/ORPO/KTO/RRHF | Preferences/desirability | Usually no | Usually no | simple, stable | may lag online RL in some settings |
| Online IPO/Online DPO | Preferences | Yes | Optional | adaptive | non-trivial infra |
| Constitutional/RLAIF | AI-evaluated principles | Varies | Often yes | scalable safety tuning | judge bias risk |
| RLVR + GRPO | Verifiable correctness | Yes | No (if rules suffice) | excellent in math/code | weak for subjective quality |

## Practical Guidance
- Default baseline: DPO-like training on high-quality preference data.
- Math/code-heavy objective tasks: RLVR + GRPO-style training.
- Safety-heavy policy constraints: add Constitutional/RLAIF components.
- If you have robust online RL infra: test PPO/REINFORCE-family against DPO baseline under equal compute.

## Competing Views / Uncertainty
- Debate: RLVR improves true reasoning vs mostly sampling efficiency.
- "RL-free beats RL" is context-dependent, not universal.
- Benchmark wins do not always transfer to product behavior.

## Media & Visual Evidence
- ![InstructGPT canonical RLHF reference](https://arxiv.org/static/browse/0.3.4/images/arxiv-logo-one-color-white.svg)
  - Source: [Training language models to follow instructions with human feedback](https://arxiv.org/abs/2203.02155)
  - Why it matters: foundational modern RLHF pipeline.

## Open Questions
- When does online preference optimization consistently outperform offline DPO-family at equal compute?
- How to robustly prevent verifier gaming in RLVR at scale?
- What hybrid objective best combines subjective preference alignment with objective correctness?

## References (Clickable)
- [R1] [Fine-Tuning Language Models from Human Preferences (2019)](https://arxiv.org/abs/1909.08593)
- [R2] [InstructGPT / RLHF (2022)](https://arxiv.org/abs/2203.02155)
- [R3] [RLHF with PPO implementation details (ICLR blog)](https://iclr-blogposts.github.io/2024/blog/the-n-implementation-details-of-rlhf-with-ppo/)
- [R4] [Direct Preference Optimization (DPO)](https://arxiv.org/abs/2305.18290)
- [R5] [ORPO: Monolithic Preference Optimization](https://arxiv.org/abs/2403.07691)
- [R6] [KTO: Prospect-theoretic objective](https://arxiv.org/abs/2402.01306)
- [R7] [RRHF: Rank Responses to Align](https://arxiv.org/abs/2304.05302)
- [R8] [Online preference optimisation (IPO/Nash-MD)](https://arxiv.org/abs/2403.08635)
- [R9] [Robust Multi-Objective Online DPO](https://arxiv.org/abs/2503.00295)
- [R10] [Back to Basics: REINFORCE-style alignment](https://arxiv.org/abs/2402.14740)
- [R11] [Constitutional AI / RLAIF](https://arxiv.org/abs/2212.08073)
- [R12] [DeepSeekMath (GRPO context)](https://arxiv.org/abs/2402.03300)
- [R13] [RLVR reasoning analysis](https://arxiv.org/abs/2506.14245)
- [R14] [OpenAI lm-human-preferences (PPO RLHF code)](https://github.com/openai/lm-human-preferences)
- [R15] [Hugging Face TRL DPO script](https://github.com/huggingface/trl/blob/main/trl/scripts/dpo.py)
- [R16] [Hugging Face TRL GRPO script](https://github.com/huggingface/trl/blob/main/trl/scripts/grpo.py)
- [R17] [Hugging Face TRL RLOO script](https://github.com/huggingface/trl/blob/main/trl/scripts/rloo.py)
- [R18] [Hugging Face TRL ORPO example](https://github.com/huggingface/trl/blob/main/examples/scripts/orpo.py)
- [R19] [Hugging Face TRL KTO example](https://github.com/huggingface/trl/blob/main/examples/scripts/kto.py)
- [R20] [GanjinZero RRHF training code](https://github.com/GanjinZero/RRHF/blob/main/train.py)

## Spoilers (Hidden Until Requested)
> Intentionally left blank. Ask to reveal spoiler analysis.

## Change Log
- 2026-04-04T19:36:21+05:30 : Finalized as Completed note and adapted display equations for GitHub Pages compatibility.
- 2026-04-04T17:05:55+05:30: Created in-progress note and started source collection plan.
- 2026-04-04T17:05:55+05:30: Added first-pass synthesis across PPO-RLHF, DPO-family, GRPO/RLVR, RLAIF, online preference methods.
- 2026-04-04T18:28:01+05:30: Added mathematical formulations and algorithm sketches.
- 2026-04-04T18:32:20+05:30: Converted LaTeX delimiters for markserv compatibility.
- 2026-04-04T18:35:47+05:30: Reorganized note so equations/algorithms live inside each relevant method section (context-preserving structure).
- 2026-04-04T18:42:28+05:30: Added per-formula `Plain English` and `Variables` blocks directly under each equation for readability and rendering-safe structure.
- 2026-04-04T19:00:33+05:30: Added per-method intuition/origin narratives and line-by-line code examples from major public repos (PPO, DPO, ORPO, KTO, RRHF, RLOO, GRPO/RLVR, plus RLAIF usage pattern).
