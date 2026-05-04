---
layout: default
title: Delegating Tasks to Smaller Models Without Losing Quality
permalink: /musings/delegating-tasks-to-smaller-models-without-losing-quality/
---

# Delegating Tasks to Smaller Models Without Losing Quality
_AI-generated research draft. Verify critical claims with primary sources._
Status: Complete
Completed: 2026-04-16T23:29:17+05:30

## TL;DR
- The best current pattern is not to replace a frontier model with a smaller one wholesale, but to **route verifiable subtasks** to smaller, cheaper, faster models while keeping the frontier model as architect, planner, and final verifier.
- The strongest academic evidence supports four mature design families: **routing-based delegation**, **spec-driven generation**, **verifier-in-the-loop**, and **multi-stage refinement**.
- For software engineering, the most robust pattern is **generate-then-verify**: a smaller model writes code, tests, or documentation; a larger model (or deterministic tools) verify output.
- Anthropic and OpenAI’s public implementation guidance converges on the same architectural advice: separate specification from generation, verify every output, and let the larger model handle ambiguity.
- A practical system for GPT-5.4 / Claude Opus + one local 3090 should treat the frontier model as the “architect” and the local model as “workers” with well-defined, independently verifiable tasks.
- The frontier of the field is shifting from task-level routing toward **step-level delegation** where a supervisor model can interrupt, redirect, and verify at intermediate steps.

## Overview
This note surveys how to delegate tasks from a large frontier model to smaller, cheaper, locally-runnable models without unacceptable quality loss. It covers the research literature, major lab guidance, inference-time techniques, and a practical system design for a developer with one frontier API key and one local GPU.

The capability gap between frontier models (GPT-4o, GPT-5, Claude Opus) and strong open-weight models (Llama-4, DeepSeek-V3, Qwen-3) is closing on many benchmarks, but not uniformly. The key insight from the literature is that quality loss is not a function of model size alone — it depends on **task decomposability** and **verifiability**.

## Research Landscape

### Routing-Based Delegation
The most mature approach. An upstream router (which can be the frontier model itself, or a lightweight classifier) decides which model handles each query or subtask.

- **Hybrid LLM (Ding et al., 2024)** showed that routing between GPT-4 and a smaller model based on query difficulty can maintain >95% of GPT-4 quality at <50% of the cost on routing-amenable benchmarks.
- **FrugalGPT (Chen et al., 2023)** demonstrated cascading through multiple LLM APIs, calling cheaper models first and escalating only when confidence is low.
- **RouteLLM (Ong et al., 2024)** open-sourced a BERT-based router that chooses between strong and weak models, achieving near-parity with GPT-4 on Chatbot Arena.
- **LLM-Blender (Jiang et al., 2023)** introduced pairwise ranking for ensemble selection — combining outputs from multiple smaller models and using a ranker to select the best.

### Spec-Driven Generation
A frontier model writes a detailed specification; a smaller model implements against it.

- **Self-Refine (Madaan et al., 2023)** showed that iterative self-feedback improves output quality for the same model, but the framework generalises to spec-writer + implementer pairs.
- **SWE-bench evaluations** indirectly demonstrate this: Claude Opus writes the plan, smaller models execute individual file changes.
- **Anthropic’s public guidance (2024–2025)** describes a “specification-first” pattern: let Opus/Claude-4 define what should happen, then let Haiku execute it.
- **OpenAI’s structured outputs with GPT-4o** (JSON mode, function calling) represent a spec-driven approach where the frontier model defines the output shape and smaller models fill it. The key: JSON schema acts as a verifiable specification.

### Verifier-in-the-Loop
Instead of trusting the smaller model’s output, run a verification step.

- **CRITIC (Gou et al., 2024)** introduced a framework where an LLM critiques its own outputs using external tools (search, code execution). This can be adapted so a smaller model generates and a larger model verifies.
- **LLM-as-Judge** approaches are well-established but have known biases (position bias, verbosity bias). Using deterministic verifiers (test suites, type checkers, linters) avoids these.
- **Constitutional AI (Bai et al., 2022)** demonstrated that a model can critique and revise its own outputs against a constitution of principles. The same pattern works cross-model: a frontier model critiques a smaller model’s output against task-specific rubrics.
- **Code generation** is the strongest domain for verifier-in-the-loop because unit tests, type checking, and linting provide deterministic (not LLM-judged) verification. The “AlphaCodium” flow (generate, test, reflect, fix) shows this explicitly.

### Multi-Stage Refinement
Break tasks into pipeline stages, using larger models only for the hardest stages.

- **ReAct (Yao et al., 2023)** and **Chain-of-Thought** reasoning are more effective with frontier models, but the tool-use steps can be offloaded.
- **PEARL (Sun et al., 2024)** introduced stage-wise delegation where a large model plans and a small model executes, with explicit handoff points.
- The emerging pattern for agentic coding (Devin, OpenHands, SWE-Agent) uses a strong model for reasoning about next actions and a weaker model for file editing and terminal commands.

## Lab Guidance

### Anthropic
- Official guidance recommends a **specification-first** pattern: use Opus for planning, requirements, and architecture; use Haiku for implementation, boilerplate, and mechanical refactoring.
- Claude’s system of Opus/Sonnet/Haiku is explicitly designed for this tiered delegation pattern.
- Key principle: **never delegate ambiguous tasks**. If the smaller model needs to interpret intent, let the larger model handle it.

### OpenAI
- GPT-4o and GPT-5 are positioned as “orchestrators” in multi-agent systems.
- Structured Outputs and function calling turn the frontier model into a specification engine that smaller models (including open-weight models) can implement against.
- The Assistants API with code interpreter follows a generate-then-verify pattern: code is generated, executed in a sandbox, and results flow back to the model.

### DeepMind / Google
- Gemini’s multi-model architecture demonstrates internal tiered routing, where different model capacities handle different parts of a prompt.
- The “mixture of dep” approach (Gemini 2.5) suggests internal routing is happening at inference time, with different expert sub-networks activating for different subtasks.

## Inference-Time Techniques That Help

These techniques apply regardless of which models you’re using and can close some of the quality gap:

- **Best-of-N sampling**: Generate N outputs from the smaller model, use the frontier model to select the best. Costs scale with N but quality approaches frontier-model single-pass quality for many tasks.
- **Majority voting / self-consistency**: For reasoning tasks, generate multiple chains of thought and take the majority answer. Reduces variance from weaker models.
- **Speculative decoding**: Use the smaller model to draft tokens, verify with the larger model. This is primarily a latency optimisation, not a quality improvement, but it enables practical systems.
- **Constrained decoding**: Use grammars, regex, or JSON schema to constrain the smaller model’s output structure. Reduces hallucination in structured generation tasks.

## Practical System Design

For a developer with:
- One frontier API key (GPT-5 or Claude Opus)
- One local GPU (3090 capable of running 8B–32B models)

### Architecture

```
[User Request]
     |
     v
[Frontier Model: Analyzer]
  - Classifies task type and difficulty
  - Decomposes into subtasks
  - Writes specifications and acceptance criteria
     |
     v
[Local Model(s): Workers]
  - Execute well-defined subtasks
  - Generate code, text, structured data
  - Each subtask has verifiable output criteria
     |
     v
[Verification Layer]
  - Deterministic: tests, type checks, linters, diff validators
  - LLM-based: frontier model reviews output
  - Retry loop: if verification fails, send back to worker with feedback
     |
     v
[Frontier Model: Assembler]
  - Combines verified outputs
  - Handles integration issues
  - Presents final result to user
```

### Task Suitability Framework

| Task Type | Delegate? | Verification Method | Expected Quality |
|-----------|-----------|-------------------|------------------|
| Write unit tests | Yes | Run tests (pass/fail) | Very high |
| Generate boilerplate | Yes | Lint + type check | Very high |
| Translate text | Yes | BLEU/comet + frontier review | High |
| Summarise document | Yes | Frontier model review | Medium–High |
| Write documentation | Yes | Frontier model review | Medium–High |
| Refactor code (mechanical) | Yes | Tests still pass | High |
| Write complex algorithm | Conditional | Tests + frontier review | Medium |
| Architecture decisions | No | — | N/A |
| Security-critical code | No | — | N/A |
| Ambiguous requirements | No | — | N/A |

### Cost Model

Assume:
- Frontier model: $15/M input tokens, $60/M output tokens (GPT-5 tier)
- Local model: effectively free (electricity only)
- Best-of-N with N=3 for verification

For a typical coding session:
- Without delegation: 100% frontier model = $X
- With delegation (80% local, 20% frontier): ~0.2X + overhead (~0.1X for verification passes) ≈ 0.3X
- Savings: 60–70% cost reduction with minimal quality loss on suitable tasks

## What Not to Delegate

- Tasks requiring **novel reasoning**: if the task requires a genuinely new insight, the smaller model will produce a plausible-sounding but wrong answer.
- Tasks requiring **long-horizon planning**: smaller models lose coherence over long contexts.
- **Security-sensitive** operations: smaller models are more susceptible to prompt injection and jailbreaking.
- Tasks with **ambiguous or underspecified** requirements: the smaller model will hallucinate to fill gaps rather than asking for clarification.
- **User-facing** generation where quality perception matters: users notice and care about small quality degradations.

## Open Questions

1. **How small is too small?** The quality cliff appears around 7B parameters for complex reasoning. 32B–70B models are much closer to frontier quality on verifiable tasks.
2. **What is the best verification strategy per task type?** The literature is converging on deterministic verification where possible, LLM-as-judge where necessary, but no systematic comparison exists.
3. **Can the router itself be a small model?** RouteLLM suggests yes for binary strong/weak decisions, but multi-way routing to specialised models is still frontier-model territory.
4. **How does this interact with agentic frameworks?** Most agentic coding systems (Devin, OpenHands) already use tiered models implicitly. Making this explicit and configurable is an active area.
5. **What about vision?** The quality gap between frontier vision models (GPT-4V, Claude Vision) and open-weight VLMs (Llava, Qwen-VL) is wider than the text gap. Vision tasks may need more frontier model involvement.

## Key Papers

| Paper | Year | Key Finding |
|-------|------|-------------|
| FrugalGPT (Chen et al.) | 2023 | Cascading LLM APIs reduces cost 50–90% |
| Hybrid LLM (Ding et al.) | 2024 | Routing maintains 95%+ quality at <50% cost |
| RouteLLM (Ong et al.) | 2024 | Open-source router for strong/weak model selection |
| LLM-Blender (Jiang et al.) | 2023 | Ensemble + ranking outperforms single model selection |
| CRITIC (Gou et al.) | 2024 | Self-critique with external tools improves smaller model output |
| Self-Refine (Madaan et al.) | 2023 | Iterative self-feedback closes quality gap |
| Constitutional AI (Bai et al.) | 2022 | Principle-based self-critique improves alignment and quality |

## Recommendations

1. **Start with deterministic verification tasks.** Code generation with test suites is the highest-ROI delegation target. Quality is measurable, and failures are caught automatically.
2. **Use Best-of-N for quality-critical outputs** where deterministic verification isn’t possible. N=3 or N=5 with frontier model selection.
3. **Always write acceptance criteria before delegating.** The spec is the contract. Without it, the smaller model will drift.
4. **Keep the frontier model in the loop for integration.** Individual subtask outputs may all verify correctly but not compose well. The frontier model should handle integration.
5. **Monitor quality real-time.** Log verification pass rates, retry counts, and frontier model intervention frequency. Degradation will be gradual, not sudden.
6. **Don’t delegate user-facing generation** unless you’re willing to have the frontier model review every output. Users tolerate less quality variance than automated pipelines.

## Sources
- Chen et al. (2023) “FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance”
- Ding et al. (2024) “Hybrid LLM: Cost-Efficient and Quality-Aware Query Routing”
- Ong et al. (2024) “RouteLLM: Learning to Route LLMs from Preference Data”
- Jiang et al. (2023) “LLM-Blender: Ensembling Large Language Models with Pairwise Ranking and Generative Fusion”
- Gou et al. (2024) “CRITIC: Large Language Models Can Self-Correct with Tool-Interactive Critiquing”
- Madaan et al. (2023) “Self-Refine: Iterative Refinement with Self-Feedback”
- Yao et al. (2023) “ReAct: Synergizing Reasoning and Acting in Language Models”
- Bai et al. (2022) “Constitutional AI: Harmlessness from AI Feedback”
- Sun et al. (2024) “PEARL: Planning and Executing Actions through Reasoning with Language”
- Anthropic public engineering blog posts on Claude model tiering (2024–2025)
- OpenAI platform documentation on structured outputs and multi-agent patterns (2024–2025)

_Generated by Hermes Agent research workflow. Final review by author. Verified 2026-04-16._
