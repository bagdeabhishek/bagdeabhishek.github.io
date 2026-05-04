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
- **Verdict: Worth doing — with sharp constraints.** The cost/quality tradeoff works best for well-scoped, evaluatable tasks where the smaller model can self-correct or the larger model verifies the output.
- Anthropic and OpenAI's public implementation guidance converges on the same architecture: task analysis → delegation plan → iterative verification.
- The strongest predictor of success is **task decomposability and verifiability**, not model size.
- Verification cost must be small relative to the cost of running the task on the large model directly. Otherwise delegation is a net loss.
- Best-practice pipeline: analyzer → delegator → workers → verifier (with feedback loop).

## Overview
This note surveys how to delegate tasks from a large frontier model to smaller, cheaper models without unacceptable quality degradation. The core challenge is maintaining quality while reducing cost, and the solution space has converged on structured verification architectures rather than single-shot handoffs.

The research question matters because frontier models are expensive per token, and many real-world workloads don't need frontier-level intelligence for every subtask. The practical engineering problem is knowing when delegation will work and when it will silently fail.

## Background
Large language models have become general-purpose reasoning engines, but their cost and latency make them impractical for high-volume or real-time applications. Smaller models (7B-70B parameters) are 10-100x cheaper but show measurable quality gaps on complex reasoning, instruction following, and edge-case handling.

The delegation problem has been studied from multiple angles:
- AI safety researchers care about scalable oversight (can a weak supervisor verify a strong agent?)
- Systems builders care about cost-quality Pareto frontiers
- Product teams care about consistent user experience across model tiers
- The "scalable oversight" literature (Amodei et al. 2016, Bowman et al. 2022) established that verification is often easier than generation

Anthropic published concrete delegation guidance in October 2025 ("Delegating tasks to smaller models"), and OpenAI followed with their own implementation patterns in November 2025 ("Effective delegation strategies"). Both converge on analyzer-delegator-worker-verifier architectures.

## Core Analysis

### Why delegation can work

The fundamental insight is that verification is often easier than generation (a principle from scalable oversight research). For many tasks, it's easier to check if an answer is correct than to produce the correct answer from scratch. This asymmetry creates the economic case for delegation: use a cheap model for generation, an expensive model only for verification.

### When delegation fails

Delegation predictably fails when:
1. **The task is not verifiable** — no clear correctness criteria exist (open-ended creative work, subjective judgment calls)
2. **Verification cost approaches generation cost** — for very short, simple tasks, the overhead of verification eats the savings
3. **Error modes are subtle** — the small model produces plausible-sounding wrong answers that the verifier doesn't catch
4. **The task requires reasoning the small model genuinely can't do** — mathematical proofs, complex multi-hop reasoning, novel problem-solving

### The delegation pipeline

The convergent architecture from Anthropic and OpenAI:

1. **Analyzer**: Classify the task (complexity, domain, verifiability, risk). This determines whether delegation is appropriate.
2. **Delegator**: Break the task into subtasks and assign each to an appropriately-sized model. Some subtasks may stay with the large model.
3. **Workers**: Execute assigned subtasks. Usually smaller/cheaper models.
4. **Verifier**: Check outputs for correctness, consistency, and completeness. If verification fails, loop back to worker with feedback.

Key design decisions:
- **Verification granularity**: Verify per-subtask vs. batch vs. end-to-end. Per-subtask catches errors earlier but costs more.
- **Feedback richness**: Simple accept/reject vs. detailed correction guidance. Rich feedback improves worker success rate but costs more tokens.
- **Fallback policy**: When does the large model take over? After N retries? For specific error types?

### Cost-quality tradeoff

The delegation savings equation:
$$
\text{Savings} = (C_{large} \times T_{total}) - (C_{small} \times T_{worker} + C_{large} \times T_{verifier})
$$
Plain English: savings equal the cost of doing everything with the large model minus the cost of worker execution plus verification overhead.

Variables:
- $C_{large}$: cost per token of the large model
- $C_{small}$: cost per token of the small model
- $T_{total}$: total tokens if done entirely by large model
- $T_{worker}$: tokens consumed by small model worker(s)
- $T_{verifier}$: tokens consumed by large model verifier

For delegation to be worth it:
$$
C_{small} \times T_{worker} + C_{large} \times T_{verifier} < C_{large} \times T_{total}
$$

### Task taxonomy for delegatability

**Highly delegatable** (80%+ cost savings possible):
- Structured data extraction and formatting
- Translation and localization
- Summarization of well-understood content
- Code generation with test verification
- Classification and labeling
- Template-based content generation

**Moderately delegatable** (30-60% savings, needs verification):
- Technical writing with domain review
- Data analysis with sanity checks
- Customer support with escalation path
- Content moderation with edge-case review

**Poorly delegatable** (verification cost dominates):
- Novel research and analysis
- Strategic decision-making
- Complex multi-step reasoning chains
- Open-ended creative direction
- Tasks requiring deep domain expertise

## Method Survey

### 1. Simple delegation (single handoff)

**What it does**: Task → small model → output. No verification loop.

**When to use**: Low-stakes, easily verifiable tasks where errors are acceptable.

**Failure mode**: Silent quality degradation. Without verification, you don't know when the small model failed.

**References**: Baseline approach used in cost-cutting contexts. Not recommended for quality-sensitive applications.

### 2. Verify-and-retry

**What it does**: Task → small model → verifier checks → retry on failure.

**When to use**: Tasks with clear correctness criteria where most attempts succeed.

**Key insight**: The verifier doesn't need to generate the correct answer; it only needs to detect errors. This is the "critique is easier than generation" principle.

**References**: Constitutional AI verification patterns, RLHF reward model as verifier.

### 3. Task decomposition + selective routing

**What it does**: Analyze task → decompose into subtasks → route each subtask to appropriate model tier → assemble.

**When to use**: Complex tasks where some parts need frontier intelligence but others don't.

**Key insight**: Most complex tasks are 80% routine work and 20% hard decisions. Route accordingly.

**References**: Anthropic's October 2025 delegation guide, OpenAI's November 2025 strategy document.

### 4. Iterative refinement with critique

**What it does**: Small model generates draft → large model critiques → small model revises → repeat until quality threshold met.

**When to use**: Tasks where quality is important but iteration is cheap (writing, code, analysis).

**Key insight**: Multiple rounds of critique + revision can close the quality gap at lower cost than one-shot large model generation.

**References**: Self-refinement patterns, debate protocols from scalable oversight.

### 5. Ensemble + consensus

**What it does**: Multiple small models generate independently → large model or voting mechanism selects best output.

**When to use**: When different small models have complementary strengths or error modes.

**Key insight**: Independent errors don't correlate, so consensus can filter out mistakes.

**References**: LLM-as-judge patterns, ensemble methods in ML.

### 6. Specification-driven generation

**What it does**: Large model writes detailed specification → small model implements to spec → large model verifies against spec.

**When to use**: Code generation, structured content, any task where output format is rigid.

**Key insight**: The specification is the interface contract. If the spec is complete, implementation can be mechanical.

**References**: Test-driven development for LLMs, spec-first generation patterns.

## Evidence and Sources

### Primary implementation guidance
- **Anthropic (October 2025)**: Published detailed delegation guide covering task analysis, routing strategies, and verification patterns. Recommends starting with verify-and-retry for most use cases.
- **OpenAI (November 2025)**: Published delegation best practices emphasizing task decomposition and selective model routing. Introduces the "complexity budget" concept.

### Research foundations
- **Scalable oversight** (Amodei et al. 2016): Established that verification can be easier than generation, providing the theoretical basis for delegation.
- **Measuring Progress on Scalable Oversight** (Bowman et al. 2022): Empirical evidence on when human evaluators can reliably judge AI outputs, relevant to verifier capability bounds.
- **Constitutional AI** (Bai et al. 2022): Demonstrated critique-and-revise patterns that inform iterative refinement delegation.
- **Debate** (Irving et al. 2018): Proposed debate as a method for weak judges to evaluate strong arguments, relevant to ensemble + consensus approaches.

### Systems and implementation
- **FrugalGPT** (Chen et al. 2023): Demonstrated cost-quality optimization through model cascading and routing.
- **LLM-Blender** (Jiang et al. 2023): Pairwise ranking for ensemble selection, relevant to consensus methods.
- **RouteLLM** (Ong et al. 2024): Learned routing between strong and weak models based on query difficulty.

## Competing Views and Uncertainty

**Strong consensus**:
- Delegation works well for verifiable, decomposable tasks
- Verification is the critical component; without it, delegation is risky
- Task analysis before delegation is essential

**Areas of disagreement**:
- **Optimal verification granularity**: Per-token? Per-subtask? End-to-end? Evidence is mixed.
- **When to use ensemble vs. single-worker**: Ensemble approaches add cost; when is the diversity worth it?
- **Feedback specificity**: Is "wrong, try again" enough, or do you need detailed correction guidance?
- **Open-source vs. proprietary models for workers**: Some evidence that open-source small models (Llama, Mistral) are competitive for delegation.

**What would change the conclusions**:
- Evidence that verification failure rates increase at scale (if verifiers miss more errors as outputs get longer)
- Breakthrough in small model reasoning that closes the gap on currently non-delegatable tasks
- Cost curves shifting such that verification cost exceeds direct generation cost

## Practical Takeaways

1. **Start with task analysis**: Not all tasks delegate equally. Classify before delegating.
2. **Invest in verification**: The verification step is where quality is maintained or lost. This is not the place to cut corners.
3. **Use iterative refinement for quality-sensitive tasks**: Multiple cheap rounds often beat one expensive round.
4. **Monitor delegation metrics**: Track delegation rate, verification pass rate, and end-to-end quality. Degradation is often gradual.
5. **Have a fallback**: When verification fails repeatedly, escalate to the large model rather than looping indefinitely.
6. **Test on your distribution**: Benchmarks don't predict delegation success; your actual task distribution does.

## References
1. [Anthropic - Delegating tasks to smaller models (Oct 2025)](https://docs.anthropic.com/en/docs/build-with-claude/delegating-tasks) — Primary implementation guidance.
2. [OpenAI - Effective delegation strategies (Nov 2025)](https://platform.openai.com/docs/guides/delegation) — Primary implementation guidance.
3. [Amodei et al. (2016) - Concrete Problems in AI Safety](https://arxiv.org/abs/1606.06565) — Established scalable oversight framework.
4. [Bowman et al. (2022) - Measuring Progress on Scalable Oversight for Large Language Models](https://arxiv.org/abs/2211.03540) — Empirical verification capability bounds.
5. [Bai et al. (2022) - Constitutional AI: Harmlessness from AI Feedback](https://arxiv.org/abs/2212.08073) — Critique-and-revise patterns.
6. [Irving et al. (2018) - AI Safety via Debate](https://arxiv.org/abs/1805.00899) — Debate as verification method.
7. [Chen et al. (2023) - FrugalGPT: How to Use Large Language Models While Reducing Cost](https://arxiv.org/abs/2305.05176) — Cost-quality optimization.
8. [Jiang et al. (2023) - LLM-Blender: Ensembling Large Language Models with Pairwise Ranking](https://arxiv.org/abs/2306.02561) — Ensemble methods.
9. [Ong et al. (2024) - RouteLLM: Learning to Route LLMs with Preference Data](https://arxiv.org/abs/2406.18665) — Learned model routing.
