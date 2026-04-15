---
layout: default
title: LLM Post-Training RL Methods
permalink: /musings/llm-post-training-rl-methods/
---

# LLM Post-Training RL Methods
_AI-assisted research synthesis. Verify critical claims with primary sources._
Status: Completed
Last updated: 2026-04-15T21:48:27+05:30
Mode: method-survey

## Summary
- Modern LLM post-training is best understood as a set of different feedback pipelines, not as one single RLHF recipe.
- Each process exists to solve a different bottleneck: SFT teaches basic behavior, reward modeling scores behavior, PPO improves policy online, DPO-family methods simplify preference optimization, and RLVR/GRPO exploit objective verifiers.
- The most important question is not "which method is most advanced?" but "what kind of signal do I actually have: demonstrations, pairwise preferences, AI critiques, or verifiable rewards?"
- PPO-style RLHF helps when you need online policy shaping against a learned reward, but it is operationally heavy.
- DPO-family methods help by turning preference learning into a simpler direct optimization problem.
- RLVR and GRPO help most when correctness is externally checkable, such as in math and code, because they replace subjective judging with programmatic reward.

## Overview
Post-training is the stage where a pretrained or instruction-tuned model is pushed toward useful behavior after generic next-token learning. In practice, this is where teams try to make a model more helpful, more aligned, better at reasoning, more controllable, or safer in deployment.

The confusing part is that many articles introduce these methods as if they are interchangeable upgrades on one ladder. They are not. Each process changes a different part of the training loop and helps for a different reason. The real topic is therefore not just a list of acronyms. It is an explanation of what each stage does to the model and why that stage exists.

## Background
A good mental model is that post-training needs three ingredients:
- a model that can already produce plausible answers
- a signal telling you which answers are better
- an optimization method that moves the model toward those better answers without breaking everything else

Different methods differ mainly in the second and third ingredients.

### The broad pipeline before method-specific details
1. Pretraining teaches broad language competence.
2. Supervised fine-tuning (SFT) teaches basic instruction-following or task format.
3. Post-training methods then use preferences, critiques, or verifiable rewards to further shape behavior.

That means many of the named methods below are not replacements for pretraining or even always for SFT. They usually sit on top of an already competent base policy.

## Core Analysis
### Problem framing
The central problem is: how do we reliably teach the model which outputs are better when "better" is expensive to label, partly subjective, and easy to game?

Different post-training methods answer that question differently:
- RLHF says: learn a reward model from preference judgments, then optimize against it.
- DPO-family says: skip the separate reward-model-plus-RL stack and optimize directly from preference comparisons.
- RLVR says: when correctness is objectively checkable, use that verifier directly.
- Constitutional or RLAIF-style methods say: scale feedback by using principle-guided AI critique instead of relying only on human rankings.

### Method families
#### Step 1: Supervised Fine-Tuning (SFT)
What it does:
SFT teaches the model to imitate high-quality answers on curated examples. It is usually the stage where a raw base model learns basic assistant behavior: answer the question, use the requested format, follow simple instructions, and stay on topic.

How it helps:
SFT creates a usable starting policy. Without it, later preference or RL stages have to optimize a model that may still answer in the wrong format, ignore instructions, or produce unstable completions. In other words, SFT does not solve nuanced alignment by itself, but it gives the later stages something worth refining.

What it does not solve well:
It mostly imitates demonstrations. That means it is weaker at teaching subtle tradeoffs like "be more helpful but not too verbose" or "prefer safer answer A over plausible but risky answer B" when those tradeoffs are not exhaustively demonstrated.

#### PPO-style RLHF
What it does:
This is the classic RLHF pipeline. It usually has four moving parts:
1. collect demonstrations to get an SFT policy
2. sample multiple model answers
3. ask humans to rank those answers
4. train a reward model on those rankings, then optimize the policy online with PPO

The core objective is:

$$
\max_\theta \; \mathbb{E}_{x, y \sim \pi_\theta(\cdot|x)}[r_\phi(x,y)] - \beta\,\mathrm{KL}(\pi_\theta(\cdot|x)\|\pi_{ref}(\cdot|x))
$$
Plain English: improve outputs that the reward model scores highly, while penalizing the model if it drifts too far from a trusted reference policy.
Variables:
- $\theta$: trainable policy parameters
- $x$: prompt
- $y$: sampled completion
- $r_\phi$: reward-model score
- $\pi_{ref}$: reference policy
- $\beta$: KL-penalty strength

What the reward model does:
The reward model turns pairwise or ranked human judgments into a reusable scoring function. Instead of asking humans to compare every future answer during optimization, you train a model that predicts which output humans would prefer.

How PPO helps:
PPO is the online optimization step that actually changes the policy using reward-model scores. It helps by letting the current model sample fresh outputs, get scored, and then update itself carefully rather than jumping too far in one step. The KL penalty is important because it prevents the policy from drifting so far toward reward maximization that it becomes weird, repetitive, or reward-hacky.

Why this process exists:
This stack exists because pairwise human preference data is rich but sparse. The reward model compresses that signal, and PPO gives you a way to optimize against it online.

What it helps with:
- nuanced preference shaping beyond simple imitation
- online refinement against current model behavior
- cases where the best answer depends on subtle human judgment rather than exact correctness

What makes it hard:
It is operationally expensive because every piece matters: data collection, reward-model quality, rollout quality, KL tuning, PPO stability, and reward hacking defenses.

#### DPO-family methods
What they do:
DPO, ORPO, KTO, and RRHF all try to learn from preference information without building the full reward-model-plus-PPO stack.

The canonical DPO loss is:

$$
\mathcal{L}_{DPO}(\theta)= -\log\sigma\Big(\beta[(\log\pi_\theta(y_w|x)-\log\pi_\theta(y_l|x))-(\log\pi_{ref}(y_w|x)-\log\pi_{ref}(y_l|x))]\Big)
$$
Plain English: increase the probability of the preferred answer relative to the rejected one, measured against a reference-model baseline.
Variables:
- $x$: prompt
- $y_w$: preferred answer
- $y_l$: rejected answer
- $\pi_\theta$: current policy
- $\pi_{ref}$: frozen reference policy
- $\beta$: scaling term

How DPO helps:
DPO helps by removing the explicit reward-model training stage and the online PPO loop. Instead of first learning a separate score function and then doing RL, it directly updates the model so preferred answers become more likely than dispreferred answers.

Why that matters:
This simplifies the training pipeline dramatically. You still use human preference data, but you no longer need to maintain as much RL machinery. That usually means fewer moving parts, lower engineering burden, and often more stable optimization.

How the variants help:
- ORPO helps by combining language-model fitting and preference pressure in one stage, reducing dependence on a separate reference-model-heavy setup.
- KTO helps when feedback is easier to express as good/bad desirability labels than as strict winner/loser pairs.
- RRHF helps when the supervision is better thought of as ranking multiple candidates rather than only pairwise binary comparisons.

What these methods are best at:
They are best when you already have preference-style data and want a simpler way to learn from it.

What they give up:
They are usually less explicitly online than PPO-style RLHF. That can matter when the model distribution shifts and you want continual updates based on fresh generations.

#### Online preference optimization variants
What they do:
These methods try to bring back some of the advantages of online RL without always paying the full PPO-style complexity cost. They sample from the current policy, score current outputs, and optimize on-policy or near-on-policy preference objectives.

How they help:
They help when static preference datasets become stale. If the current model has changed enough, old offline comparisons may no longer reflect the errors or opportunities the model now produces. Online methods keep the training signal closer to the model's current behavior.

Why they are not always the default:
They are operationally heavier than purely offline preference learning, so the extra adaptability only pays off when fresh on-policy data actually matters.

#### RLVR and GRPO
What RLVR does:
RLVR means reinforcement learning with verifiable rewards. Instead of asking humans or a learned reward model which answer is better, you use an external checker: unit tests, exact answer matching, theorem verification, parsers, or similar automatic scoring rules.

What that changes:
It changes the source of truth. The training signal is no longer "what humans preferred" but "what the verifier judged correct."

What GRPO does:
GRPO is one way to stabilize optimization in this setting by comparing multiple sampled answers for the same prompt and assigning them relative advantage.

$$
A_i = \frac{r_i - \mu_r}{\sigma_r + \varepsilon}
$$
Plain English: each sampled answer is judged relative to the other sampled answers for the same prompt, so answers that score above the group average get positive learning signal.
Variables:
- $r_i$: reward for candidate $i$
- $\mu_r$: group mean reward
- $\sigma_r$: group reward standard deviation
- $\varepsilon$: stabilizer term
- $A_i$: relative advantage for candidate $i$

How RLVR helps:
It helps by replacing expensive subjective judgment with objective feedback in domains where correctness is externally checkable. That is why it matters so much for math, code, and some structured reasoning tasks.

How GRPO helps:
It helps reduce variance and makes the update signal more comparative. Instead of asking whether an absolute score is good enough in the abstract, it asks which of the sampled answers for this prompt did better and by how much.

What this process is good at:
- pushing the model toward answers that pass tests or exact checks
- exploiting large-scale cheap automatic reward
- training reasoning systems in domains where correctness has a crisp notion

What it does not solve well:
It is much weaker for things like tone, harmlessness, style, empathy, or other qualities that do not have robust external verifiers.

#### Constitutional / RLAIF-style methods
What they do:
These methods use a written set of principles or constitutional rules to generate critiques, revisions, or preference labels with AI assistance. Instead of relying entirely on human raters, the system uses principle-guided model judgment to scale supervision.

How they help:
They help when the bottleneck is not raw optimization but feedback generation. If human labeling is too slow or expensive, AI-generated critique can create more training signal.

Why this matters:
Safety and policy alignment often require many subtle judgments. Constitutional or RLAIF-style pipelines help scale those judgments.

What the catch is:
Their quality depends heavily on the constitution, the judge model, and the critique process. If those are flawed, the system can scale biased supervision rather than good supervision.

### Representative methods
A useful way to map the families is by the main bottleneck they solve:
- SFT: teaches basic assistant behavior and formatting
- PPO-RLHF: improves behavior using a learned human-preference reward and online policy updates
- DPO/ORPO/KTO/RRHF: improve preference alignment with simpler direct objectives
- Online preference methods: keep preference optimization matched to current model behavior
- RLVR/GRPO: improve performance where objective correctness can be checked automatically
- Constitutional/RLAIF: scale critique and policy feedback when human supervision is scarce

### Tradeoffs
The main tradeoff is signal quality versus pipeline complexity.

PPO-RLHF is attractive when subtle human judgment matters and online policy shaping is worth the cost. DPO-family methods are attractive when you want much of that alignment benefit with less operational complexity. RLVR is attractive when the task has a strong verifier, because the reward is cheaper and more objective. Constitutional methods are attractive when the hard part is generating enough feedback rather than optimizing against it.

Another useful distinction is what each method assumes is available:
- SFT assumes good demonstrations.
- PPO-RLHF assumes human rankings plus infrastructure for reward modeling and online RL.
- DPO-family assumes preference-style comparisons or related label structures.
- RLVR assumes a reliable verifier.
- Constitutional methods assume a credible rule set and judge process.

Established vs inferred:
- Established: PPO-style RLHF is complex but powerful; DPO simplifies preference optimization; RLVR is well matched to verifier-rich domains.
- Inferred: RLVR improves underlying reasoning in a broad sense rather than mostly improving search or sampling efficiency. That claim is promising but still under active debate.

### Practical guidance
If you want a simple decision rule, choose the method by the strongest signal you actually trust.

- Start with SFT because later stages work better when the model already knows the basic job.
- Use PPO-style RLHF when nuanced human preference is central and you can support the heavier online pipeline.
- Use DPO-family methods when you have preference data but want a simpler and more stable optimization route.
- Use RLVR/GRPO when you have genuine external correctness checks.
- Use constitutional or RLAIF-style methods when you need scalable policy or safety feedback.

### Open questions
The field is still unresolved on several fronts:
- When do online preference methods beat offline direct-preference methods at equal compute and equal data quality?
- How much of RLVR's apparent reasoning gain is deeper reasoning versus stronger search, reranking, or sample efficiency?
- What is the best hybrid recipe for combining subjective helpfulness signals with objective correctness signals?

## Evidence and Sources
### Claim cluster 1: the post-training stack is modular because each process solves a different bottleneck
- InstructGPT is the clearest primary source for the demonstration -> preference -> reward model -> PPO pipeline.
- DPO is the clearest primary source for the claim that preference optimization can often be simplified into a direct objective.
- RLVR papers support the claim that verifier-based training is a distinct regime, not just a small variant of RLHF.

### Claim cluster 2: PPO-style RLHF helps with nuanced preference shaping, but at high operational cost
- Ouyang et al. (2022) support the usefulness of RLHF for following user intent and improving preference-aligned behavior.
- The ICLR implementation-details write-up supports the operational claim that PPO-RLHF involves many practical knobs and failure modes.

### Claim cluster 3: DPO-family methods help by simplifying preference learning
- Sharma et al. (2023/2024) support the core argument that direct preference optimization can match or beat PPO-based RLHF in some settings while being substantially simpler.
- ORPO, KTO, and RRHF support the broader trend toward lighter-weight preference optimization.

### Claim cluster 4: RLVR/GRPO helps most when correctness is checkable
- DeepSeekMath and later RLVR analysis support the claim that verifier-based feedback is especially useful in math and code.
- The stronger claim that RLVR broadly improves reasoning quality remains more debated than the narrower claim that it helps on verifier-rich tasks.

## Uncertainties and Competing Views
High-confidence claims:
- SFT, RLHF, DPO-family methods, and RLVR solve different problems in the overall training pipeline.
- PPO-style RLHF has more moving parts than DPO-family methods.
- RLVR is best matched to domains with robust verifiers.

Medium-confidence claims:
- Online preference optimization is worth the additional complexity in many production settings.
- RLVR improves reasoning itself, not just answer selection efficiency.

Competing views:
- Some researchers see frontier reasoning progress as evidence that online RL with verifiable reward is becoming central again.
- Others argue that for many real assistants, better data and cleaner preference optimization matter more than elaborate RL stacks.

What evidence would change the conclusion:
- More controlled apples-to-apples comparisons between PPO-style RLHF, DPO-family training, and RLVR under equal compute would sharpen when each process truly helps most.
- Better studies on verifier gaming would show whether some RLVR gains are more fragile than they currently appear.

## Practical Takeaways
- Think of post-training as a pipeline of jobs, not a contest of buzzwords.
- Ask first what supervision signal you trust: demonstrations, human comparisons, AI critiques, or automatic verifiers.
- Use SFT to establish basic assistant behavior, then choose later stages based on what kind of improvement you need.
- Use PPO-style RLHF when you need online preference shaping, DPO-family methods when you want simpler preference learning, and RLVR when correctness can be checked mechanically.
- When reading new post-training papers, always ask: what stage of the pipeline does this method replace, simplify, or strengthen?

## References
1. [Ziegler et al. (2019), Fine-Tuning Language Models from Human Preferences](https://arxiv.org/abs/1909.08593) — Primary; early modern RLHF formulation.
2. [Ouyang et al. (2022), Training language models to follow instructions with human feedback](https://arxiv.org/abs/2203.02155) — Primary; canonical InstructGPT RLHF pipeline.
3. [Huang et al. (2024), The N Implementation Details of RLHF with PPO](https://iclr-blogposts.github.io/2024/blog/the-n-implementation-details-of-rlhf-with-ppo/) — Secondary technical synthesis; operational complexity and implementation pitfalls.
4. [Sharma et al. (2023/2024), Direct Preference Optimization: Your Language Model is Secretly a Reward Model](https://arxiv.org/abs/2305.18290) — Primary; core DPO argument and objective.
5. [Hong et al. (2024), ORPO: Monolithic Preference Optimization without Reference Model](https://arxiv.org/abs/2403.07691) — Primary; one-stage preference optimization variant.
6. [Ethayarajh et al. (2024), KTO: Model Alignment as Prospect Theoretic Optimization](https://arxiv.org/abs/2402.01306) — Primary; desirability-label alternative to pairwise-only framing.
7. [Yuan et al. (2023), RRHF: Rank Responses to Align Language Models with Human Feedback](https://arxiv.org/abs/2304.05302) — Primary; ranking-based alignment objective.
8. [Xu et al. (2024), Online Preference Optimization for Language Model Alignment](https://arxiv.org/abs/2405.19107) — Primary; representative online preference optimization reference.
9. [Guo et al. (2024), DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models](https://arxiv.org/abs/2402.03300) — Primary; important GRPO-context reference.
10. [Zheng et al. (2025), Reinforcement Learning with Verifiable Rewards Implicitly Incentivizes Correct Reasoning in Base LLMs](https://arxiv.org/abs/2506.14245) — Primary; argues RLVR can improve reasoning quality, not just sample efficiency.
11. [Bai et al. (2022), Constitutional AI: Harmlessness from AI Feedback](https://arxiv.org/abs/2212.08073) — Primary; key RLAIF / constitutional framing.
