---
layout: default
title: Delegating Tasks to Smaller Models Without Losing Quality
permalink: /musings/delegating-tasks-to-smaller-models-without-losing-quality/
---

# Delegating Tasks to Smaller Models Without Losing Quality
_AI-assisted research synthesis. Verify critical claims with primary sources._
Status: Completed
Last updated: 2026-04-16T23:29:17+05:30
Mode: method-survey

## Summary
- The best current pattern is not to replace a frontier model with a smaller one, but to **split labor**: the stronger model plans, arbitrates, and handles edge cases, while the smaller model handles bounded, repetitive, or easily verified work.
- The strongest academic evidence supports four mature design families: **routing**, **cascaded escalation**, **generate-then-verify**, and **task-specific distillation**. These methods can materially reduce cost while preserving quality when deferral and verification are well designed.
- For software engineering, the most robust pattern is **generate-then-verify**: let the small model explore the repo, draft patches or tests, and perform first-pass review; then gate acceptance with executable checks and selective frontier-model review.
- Anthropic and OpenAI's public implementation guidance converges on the same architecture: **smaller/faster subagents for bounded work**, stronger models for coordination and ambiguous reasoning, plus heavy use of caching and evaluation.
- A practical system for GPT-5.4 / Claude Opus + one local 3090 should treat the local model as a specialist worker, not a general autonomous engineer. The best first specializations are repo exploration, context compression, diff summarization, test drafting, triage, and first-pass review.
- The frontier of the field is shifting from task-level routing toward **step-level** and **token-level** collaboration, where the larger model intervenes only on difficult subproblems or verifies speculative work from the smaller model.

## Overview
This note examines a now-common deployment problem: how to use smaller models to reduce inference cost and latency without quietly degrading quality. The problem appears in many forms—consumer chat, RAG, agent systems, code review, software repair, and long-horizon planning—but it becomes especially consequential in software engineering, where routine work is abundant and silent regressions are expensive.

The practical question is not whether smaller models can do useful work. They clearly can. The harder question is how to integrate them into a system in a way that preserves the quality bar associated with frontier models such as GPT-5.4 and Claude Opus, while also exploiting local models that can run cheaply on a single RTX 3090.

The literature and provider documentation now point to a coherent answer. Quality-preserving delegation is fundamentally a **systems problem**, not just a model-choice problem. It depends on architecture, evaluation, calibration, and verification as much as on raw model capability.

## Background
Modern model stacks face a widening performance-cost gap. Frontier models are excellent at coding, long-context synthesis, tool use, and ambiguous reasoning, but they are expensive and can be wasteful when applied to predictable or repetitive subroutines. At the same time, smaller or local models have improved enough that they can handle many support tasks well—especially when those tasks are narrow, repetitive, or structurally verifiable.

This makes delegation attractive, but only under discipline. A weak system delegates too aggressively and quietly accumulates defects. A strong system narrows the smaller model's scope, tracks when to defer, and validates outputs before acting on them.

A useful mental model is:

**frontier model = strategist / arbiter / escalation path**  
**small model = bounded worker / drafter / scanner / classifier**  
**quality = preserved by verification + routing + escalation + evals**

That mental model is more faithful to current best practice than the more dramatic idea of a fully autonomous swarm of cheap models replacing a single strong one.

## Core Analysis
### Problem framing
The deployment problem has three competing objectives:

1. **Quality** — keep output quality close to the frontier model's quality on the tasks that matter.
2. **Cost and latency** — reduce token spend, wall-clock latency, and infrastructure overhead.
3. **Operational reliability** — avoid silent failure modes, especially in software workflows where regressions may only appear later.

A compact routing objective can be written as:

$$
\pi^*(x) = \arg\max_{m \in \mathcal{M}} \Big[ U(q(x,m)) - \lambda C(x,m) - \mu L(x,m) \Big]
$$

Plain English: for a task $x$, choose the model $m$ that gives the best quality-adjusted utility after penalizing cost and latency.

Variables:
- $x$: the incoming task or prompt.
- $\mathcal{M}$: the set of candidate models or model pipelines.
- $q(x,m)$: the quality of model $m$ on task $x$.
- $U(\cdot)$: a utility function over quality.
- $C(x,m)$: the cost of using model $m$ on task $x$.
- $L(x,m)$: the latency of using model $m$ on task $x$.
- $\lambda$: how strongly the system penalizes cost.
- $\mu$: how strongly the system penalizes latency.

This formulation is useful because it makes the central point explicit: the right model is not the strongest model, but the model or cascade that best optimizes the full objective for this task.

### Pipeline mental model
A quality-preserving delegation stack usually follows this pipeline:

1. **Cheap pre-processing**  
   Classical code and systems heuristics run first: diff stats, file ownership maps, import graphs, cached context, retrieval filters, duplicate suppression, and prompt caching.

2. **Small-model bounded work**  
   The smaller model explores, summarizes, classifies, drafts, or proposes candidate actions.

3. **Verifier / deferral layer**  
   Tests, type checks, linters, semantic filters, or a stronger model determine whether the small-model output is acceptable.

4. **Frontier arbitration**  
   The stronger model sees only the focused bundle: task, candidate output, verifier results, and minimal supporting context.

5. **Learning loop**  
   Outcomes are logged and turned into evals, routing thresholds, or distillation data.

This pipeline is important because it turns delegation from a leap of faith into a measurable decision system.

### Method families
#### 1. Query-level routing
This is the simplest family. A router decides whether a prompt should go to a strong model or a weak model before any substantial work begins.

**What it does**
- Predicts task difficulty or expected quality gap.
- Sends simple tasks to the cheap model and hard tasks to the expensive one.

**How it helps**
- Easy to deploy.
- Reduces expensive-model usage on routine traffic.

**Why it exists**
- Cost differences between models are now large enough that static one-model serving is often economically irrational.

**What it is good at**
- High-volume heterogeneous request streams.
- Support, classification, Q&A, and broad tool-routing stacks.

**What it does not solve well**
- Hard tasks misclassified as easy.
- Silent failure in generative tasks where confidence is poorly calibrated.

**Representative evidence**
- **FrugalGPT** organized the problem around prompt adaptation, approximation, and cascades, showing that orchestration can beat naive best-model defaulting.
- **Hybrid LLM** showed that routing based on query difficulty and user quality target can reduce large-model calls without hurting quality.
- **RouteLLM** learned routers from preference data, showing that routing can deliver large cost reductions and can transfer across model pairs.

#### 2. Cascaded escalation
This family lets the small model try first and escalates only if the case looks difficult.

**What it does**
- A smaller model handles most cases.
- An escalation policy sends ambiguous cases onward.

**How it helps**
- Preserves low average cost while keeping a fallback path.

**Why it exists**
- Many workloads are long-tailed: most tasks are easy, but a minority are much harder.

**What it is good at**
- Triage, broad retrieval pipelines, bug localization, and repo exploration.

**What it does not solve well**
- Requires a good deferral policy.
- Raw confidence tends to be unreliable.

The strongest caution here comes from the generative-cascade literature: naive sequence-level confidence is often length-biased and poor at identifying when to defer. **Language Model Cascades: Token-level uncertainty and beyond** is especially useful because it shows that token-level uncertainty and richer deferral rules outperform naive sequence uncertainty for generation.

#### 3. Generate-then-verify
This is the most robust pattern for software engineering.

**What it does**
- The smaller model generates a candidate patch, test suite, summary, or review comment.
- A verifier stack checks it.
- The larger model handles only failures, edge cases, or final review.

**How it helps**
- Exploits the fact that many software artifacts are executable or statically analyzable.

**Why it exists**
- Code changes are often easier to verify than to derive.
- Verification can be more reliable than trying to ask a weaker model to know its own limits.

**What it is good at**
- Patch drafting.
- Test generation.
- Mechanical refactoring.
- Diff analysis.
- First-pass review.

**What it does not solve well**
- Cases where tests are weak or unavailable.
- Architectural regressions, subtle concurrency bugs, or security issues that elude automated checks.

The software-engineering literature strongly supports this pattern. **Assured LLM-Based Software Engineering** advocates a generate-and-test workflow with semantic filters, explicitly framing the problem as how to let models improve code without regressing properties or accepting hallucinated improvements.

#### 4. Task-specific distillation
This family does not merely route between models. It teaches a smaller model to imitate the stronger model on a specific task.

**What it does**
- Uses a frontier model as teacher.
- Collects outputs or trajectories.
- Fine-tunes a smaller model for narrow, repeated work.

**How it helps**
- Produces a smaller specialist that is faster and cheaper than the teacher.

**Why it exists**
- Many production tasks are narrow and repeat thousands of times.
- A small specialist often beats a generic small model on that narrow domain.

**What it is good at**
- Classification, formatting, triage, summarization, structured extraction, repetitive code chores.

**What it does not solve well**
- Broad autonomy.
- Unstable or rapidly changing task distributions.

OpenAI's public optimization guidance explicitly supports this route. Their fine-tuning and distillation docs recommend taking a prompt that works well on a larger model, collecting successful outputs, and training a smaller model to mimic them. The OpenAI Cookbook distillation example shows a distilled `gpt-4o-mini` nearly matching `gpt-4o` on a narrow classification task after training on teacher outputs.

#### 5. Step-level selective intervention
This is a more frontier pattern. Instead of routing the whole task, the larger model intervenes only on hard steps.

**What it does**
- Lets the smaller model handle most reasoning or generation steps.
- Invokes a stronger model only for uncertain or failed steps.

**How it helps**
- Preserves more savings than task-level escalation.
- Targets frontier compute at genuinely hard nodes.

**Why it exists**
- Many tasks contain a mix of easy and hard subtasks.

**What it is good at**
- Math/coding pipelines where some steps are routine and a few are brittle.

**What it does not solve well**
- Requires more sophisticated instrumentation and step-level confidence signals.

Recent papers such as **SMART**, **Route-and-Reason**, and **Router-R1** point in this direction. They are early signs that the field is moving from "which model answers?" toward "which parts require the strongest model?"

#### 6. Token-level collaboration and speculative decoding
This is the cleanest theoretical example of small-large tandem collaboration.

**What it does**
- A small draft model proposes several next tokens.
- The large target model verifies them.
- The final output is guaranteed to match the target model's distribution.

**How it helps**
- Speeds up inference without quality loss.

**Why it exists**
- Autoregressive decoding wastes hardware by generating one token per target forward pass.

**What it is good at**
- Inference acceleration.
- High-throughput serving.

**What it does not solve well**
- General task decomposition for agents.

This family matters because it demonstrates the strongest form of "small model works, large model verifies." NVIDIA's official writing on speculative decoding and TensorRT-LLM makes the point crisply: a draft-target pattern can improve throughput while preserving output quality because the target model remains authoritative.

### Representative methods
#### FrugalGPT
FrugalGPT is one of the clearest early papers to make the problem operational. Its main contribution is not a single clever algorithm but a production-oriented reframing: LLM efficiency should be thought of in terms of **prompt adaptation**, **approximation**, and **cascades**. Its headline result—that orchestrated combinations can sometimes match the strongest model at dramatically lower cost—still shapes how many later systems are framed.

#### RouteLLM
RouteLLM is especially important because it pushes routing beyond simple heuristics. The key idea is to train the router using preference data, not just benchmark labels. That matters because many real prompts are open-ended and don't have a neat ground-truth answer. The transfer result is also important: if the router generalizes across strong/weak model pairs, then the routing logic may remain useful even as the backend model pool changes.

#### Cascade-Aware Training
Cascade-Aware Training makes an underappreciated point: the small model should not be optimized in isolation if it will operate inside a cascade. It should be trained with awareness of downstream capabilities and deferral behavior. This is highly relevant to any local-model project: if you fine-tune your 3090 model, you should train it for its role in the stack, not just for generic standalone accuracy.

#### Assured LLM-Based Software Engineering
This is the most directly relevant software-engineering framing in the source set. It argues that autonomous code improvement becomes much safer when every candidate change is passed through semantic filters and measurable non-regression checks. In other words, quality comes from the harness, not from model eloquence.

### Provider implementation strategies
#### Anthropic
Anthropic's public implementation pattern is revealing. Their documentation and engineering posts consistently separate a stronger coordinator from cheaper workers.

- Model guidance explicitly places cheaper Claude variants in high-volume and sub-agent roles.
- Claude Code supports subagents with model-specific prompts and tool scopes.
- Their engineering post on the Research system describes a lead-agent plus parallel-subagent pattern, where the gains come from context separation, parallelism, and better token spending—not from some mystical collective intelligence.

That post is instructive for another reason: Anthropic openly notes that multi-agent systems spend many more tokens than a single chat interaction. This is a crucial warning. Delegation is not automatically efficient. It only pays off when the extra tokens buy better decomposition, parallelism, or verification.

Anthropic's policy work on trustworthy agents adds another important layer: failures often come from the harness, tool design, or runtime environment rather than the model alone. This supports a systems view of delegation.

#### OpenAI
OpenAI's public implementation guidance points to the same structure:

- Codex supports explicit subagents and recommends stronger defaults for coordination and smaller models for bounded worker roles.
- Prompt caching is automatic and highly emphasized, which matters because caching often saves more money than fancy routing logic at the beginning.
- OpenAI's optimization stack revolves around **evals → prompts → fine-tuning/distillation**, which is exactly the correct order for any serious delegation system.

OpenAI's open-weight `gpt-oss-20b` is also noteworthy. It is explicitly positioned for lower-latency local or specialized use cases, and it supports structured outputs, reasoning effort control, and agentic tooling. That makes it a plausible local specialist in a tandem system.

#### NVIDIA
NVIDIA's contribution is more on the infrastructure side. Their speculative decoding and TensorRT-LLM work shows how a draft-target system can achieve substantial throughput gains while preserving target-model output quality. This is directly relevant if your project eventually pushes toward serving-level tandem inference or token-level collaboration.

### Code-backed implementation patterns
#### Example 1: OpenAI-style custom worker in Codex
Repo / docs: OpenAI Codex docs  
Why this matters: it shows the production pattern of a stronger main agent delegating bounded work to a cheaper specialist.

```toml
L1: name = "docs_researcher"  # Define a dedicated bounded worker.
L2: description = "Use the docs MCP server to confirm APIs and exact references."  # Make delegation trigger explicit.
L3: model = "gpt-5.4-mini"  # Pin the worker to a cheaper/faster model.
L4: model_reasoning_effort = "medium"  # Keep effort bounded for lightweight analysis.
L5: sandbox_mode = "read-only"  # Restrict actions to safe exploration.
L6: developer_instructions = "Use docs MCP, return concise answers, do not make code changes."  # Constrain the worker's role.
```

The pattern is important: **bounded model, bounded tools, bounded goal**. That is how quality and cost are jointly preserved.

#### Example 2: OpenAI distillation workflow
Repo / docs: OpenAI Cookbook distillation example  
Why this matters: it shows how to build a smaller specialist from larger-model outputs.

```python
L1: response = client.chat.completions.create(  # Call the teacher model.
L2:     model="gpt-4o",  # Use the stronger model as the source of training traces.
L3:     messages=messages,  # Run the task with the tuned prompt.
L4:     store=True,  # Persist completions so they can later be distilled.
L5:     metadata={"distillation": "wine-distillation"},  # Tag the dataset for later filtering.
L6: )  # The stored, high-quality outputs become fine-tuning data for a smaller model.
```

The important pattern is not the specific dataset. It is the workflow: **get the strong prompt right first, collect only good traces, then distill**.

#### Example 3: Local OpenAI-family specialist via `gpt-oss`
Repo / docs: `openai/gpt-oss`  
Why this matters: it shows that a local model can be wired into a broader coding stack as a specialist worker.

```toml
L1: [model_providers.local]  # Register a local inference provider.
L2: name = "local"  # Give the provider a stable name.
L3: base_url = "http://localhost:11434/v1"  # Point Codex-compatible tooling at the local server.
L4: [profiles.oss]  # Create a profile for the open-weight worker.
L5: model = "gpt-oss:20b"  # Use the local 20B open-weight specialist.
L6: model_provider = "local"  # Route the workload to local inference.
```

This matters because it makes the hybrid stack concrete: local worker, frontier coordinator.

### Tradeoffs
The main tradeoffs are not mysterious.

#### What smaller models do well
- Retrieval-like scanning.
- Repetitive transforms.
- Diff summarization.
- First-pass triage.
- Task-specialized generation.
- Low-cost parallel exploration.

#### What stronger models still dominate
- Ambiguous planning.
- Final arbitration.
- Cross-file semantic synthesis.
- Architectural reasoning.
- Edge-case debugging.
- Subtle code review, especially when tests are weak.

#### Where teams usually go wrong
- Using a weak model as if it were a full replacement for the strong one.
- Trusting raw confidence or self-reported certainty.
- Skipping evals and "just trying it."
- Letting the frontier model reread all intermediate outputs, erasing the savings.
- Using only LLM-as-judge without executable checks.
- Training a local model for broad generality instead of narrow leverage.

### Practical guidance
#### Best practical architecture for GPT-5.4 / Opus + one local 3090
A realistic system on your stack should look like this:

**Coordinator (frontier model)**
- Task planning.
- Deciding delegation structure.
- Reviewing uncertain outputs.
- Final pass on high-risk actions.

**Local specialist (3090)**
- Repo exploration and file ranking.
- Context compression and diff summarization.
- Test drafting.
- Candidate patch drafting for routine changes.
- Issue triage.
- First-pass review comments.

**Verifier layer**
- Tests.
- Type checks.
- Linters.
- Static analyzers.
- Semantic filters.
- Optional model-based judge.

**Learning loop**
- Track which delegated tasks are accepted, repaired, or rejected.
- Turn accepted traces into local fine-tuning or distillation data.
- Turn failures into eval cases.

#### Best first specializations for the local model
If you want immediate leverage, do not start with "local autonomous coding agent." Start with these:

1. **repo explorer / file ranker**
2. **diff summarizer**
3. **test drafter**
4. **first-pass code reviewer**
5. **issue / bug triager**
6. **mechanical refactor worker**

These are predictable enough to evaluate and frequent enough to justify specialization.

#### Local model choices for one 3090
Based on current official model pages and practical fit:

- **Qwen2.5-Coder-14B / 32B** — best coding-oriented local worker family from the currently inspected sources.
- **DeepSeek-R1-Distill-Qwen-14B / 32B** — attractive if you want a more reasoning-heavy local reviewer or discriminator.
- **gpt-oss-20b** — strong local OpenAI-family option for specialized or agentic bounded work.

A sensible near-term pairing is:
- `Qwen2.5-Coder-14B` or `32B` for generation/exploration,
- `DeepSeek-R1-Distill-Qwen-14B` for reviewer/discriminator experiments,
- with GPT-5.4 or Claude Opus as final arbiter.

### Open questions
Several research questions remain genuinely open.

1. **How much should routing rely on learned confidence versus external verification?**  
   Current evidence suggests external verification is safer, especially in software.

2. **Can step-level intervention beat task-level routing in real engineering workflows?**  
   Likely yes for some domains, but operational complexity rises sharply.

3. **Can a small local model be trained to defer well, not just answer well?**  
   Cascade-aware training suggests yes, but this is underexplored in coding-specific stacks.

4. **What is the right role for LLM-as-judge in software?**  
   It seems useful as a secondary signal, but not yet as a sole gate.

5. **How should one evaluate delegation in software engineering?**  
   Benchmark success is not enough. You need repository-specific evals that measure regressions, missed bugs, reviewer usefulness, and escalation efficiency.

## Evidence and Sources
### Routing and cascades
- **FrugalGPT** — framed the cost problem in terms of prompt adaptation, approximation, and cascades.
- **Hybrid LLM** — query-difficulty routing with a tunable quality target.
- **RouteLLM** — preference-trained routers with strong cost-quality tradeoffs.
- **Token-level uncertainty and beyond** — showed why generative deferral is harder than classification deferral.
- **Cascade-Aware Training** — argued that the small model should be trained for its role in the cascade, not in isolation.

### Verification and software engineering
- **Assured LLM-Based Software Engineering** — generate-and-test plus semantic filters.
- **LLM-as-a-Judge in Software Engineering** — evidence that model-based judgment can correlate with humans, but is not a full substitute.

### Industrial implementation patterns
- **Anthropic engineering post on multi-agent research** — orchestrator-worker architecture, gains from token budget and parallel context windows, explicit warnings about cost.
- **Anthropic trustworthy agents post** — emphasized that harnesses, tools, and environments are part of agent quality and safety.
- **OpenAI docs and cookbook** — subagents, evals, caching, and distillation as core operational primitives.
- **NVIDIA speculative decoding** — strongest industrial example of quality-preserving small+large collaboration.

## Uncertainties and Competing Views
The main uncertainty is not whether delegation works at all, but how aggressive the delegation should be.

A more optimistic view is that stronger local reasoning models and better routers may soon let most predictable engineering work stay local, with frontier models stepping in only occasionally.

A more skeptical view is that many tasks that look predictable are secretly architecture-sensitive, and that strong benchmarks can hide costly silent failures. This skeptical view is especially persuasive in security review, concurrency, data migrations, and production incident response.

Another uncertainty is whether the future belongs more to **trained routers** or to **verification-heavy pipelines**. The current evidence suggests that for software engineering, verification-heavy pipelines are the safer first move, while learned routing becomes more valuable after you have strong repository-specific eval data.

## Practical Takeaways
- Treat smaller models as **specialists**, not miniature general replacements for frontier models.
- In software engineering, start with **generate-then-verify**, not pure routing.
- Exhaust **prompt caching**, context pruning, and cheap classical heuristics before building complicated routers.
- Build **evals first**. Delegation without measurement is just hidden technical debt.
- Fine-tune or distill narrow local specialists only after you collect accepted traces from a stronger model.
- If you want to push the frontier, work on **selective repair** or **step-level intervention**, not just a one-shot router.

## References
1. [Dohan et al., Language Model Cascades (2022)](https://arxiv.org/abs/2207.10342) — unifying frame for multi-step LM composition.
2. [Chen, Zaharia, Zou, FrugalGPT (2023)](https://arxiv.org/abs/2305.05176) — prompt adaptation, approximation, and cascade framing.
3. [Ding et al., Hybrid LLM: Cost-Efficient and Quality-Aware Query Routing (ICLR 2024)](https://openreview.net/forum?id=02f3mUtqnM) — small/large routing by query difficulty.
4. [Ong et al., RouteLLM (2024/2025)](https://arxiv.org/abs/2406.18665) — preference-trained router models.
5. [Gupta et al., Language Model Cascades: Token-level uncertainty and beyond (2024)](https://arxiv.org/abs/2404.10136) — learned deferral for generative cascades.
6. [Wang et al., Cascade-Aware Training of Language Models (2024)](https://arxiv.org/abs/2406.00060) — train the small model for its cascade role.
7. [Alshahwan et al., Assured LLM-Based Software Engineering (2024)](https://arxiv.org/abs/2402.04380) — generate-and-test with semantic filters.
8. [Wang et al., Can LLMs Replace Human Evaluators? An Empirical Study of LLM-as-a-Judge in Software Engineering (2025)](https://arxiv.org/abs/2502.06193) — judge-model evidence in software engineering.
9. [Anthropic, Choosing the right model](https://docs.anthropic.com/en/docs/about-claude/models/choosing-a-model) — official model-role guidance.
10. [Anthropic, Claude Code subagents docs](https://docs.anthropic.com/en/docs/claude-code/sub-agents) — bounded subagent pattern.
11. [Anthropic, Prompt caching docs](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) — cost baseline before routing complexity.
12. [Anthropic, How we built our multi-agent research system](https://www.anthropic.com/engineering/built-multi-agent-research-system) — orchestrator-worker design and practical lessons.
13. [Anthropic, Trustworthy agents in practice](https://www.anthropic.com/research/trustworthy-agents?curius=1184) — harness/tool/environment safety framing.
14. [OpenAI, Prompt caching guide](https://platform.openai.com/docs/guides/prompt-caching) — automatic prefix caching for repeated prompts.
15. [OpenAI, Working with evals](https://platform.openai.com/docs/guides/evals) — evaluation as reliability primitive.
16. [OpenAI, Supervised fine-tuning / distillation guide](https://platform.openai.com/docs/guides/supervised-fine-tuning) — teacher-student workflow.
17. [OpenAI Cookbook, Leveraging model distillation to fine-tune a model](https://developers.openai.com/cookbook/examples/leveraging_model_distillation_to_fine-tune_a_model) — practical distillation example.
18. [OpenAI, Codex subagents docs](https://developers.openai.com/codex/subagents) — explicit bounded worker configuration.
19. [OpenAI, gpt-oss repository](https://github.com/openai/gpt-oss) — official open-weight local/specialized models.
20. [Qwen, Qwen2.5-Coder-32B-Instruct model card](https://huggingface.co/Qwen/Qwen2.5-Coder-32B-Instruct) — local coding model candidate.
21. [DeepSeek, DeepSeek-R1-Distill-Qwen-14B model card](https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Qwen-14B) — local distilled reasoning model candidate.
22. [NVIDIA, Speculative decoding introduction](https://developer.nvidia.com/blog/an-introduction-to-speculative-decoding-for-reducing-latency-in-ai-inference/) — draft-target verification explanation.
23. [NVIDIA, TensorRT-LLM speculative decoding throughput](https://developer.nvidia.com/blog/tensorrt-llm-speculative-decoding-boosts-inference-throughput-by-up-to-3-6x/) — infrastructure evidence for tandem inference.
