---
layout: default
title: Delegating Tasks to Smaller Models Without Losing Quality
permalink: /musings/delegating-tasks-to-smaller-models-without-losing-quality/
---

# Delegating Tasks to Smaller Models Without Losing Quality
_AI-assisted research synthesis. Verify critical claims with primary sources._
Status: Completed
Last updated: 2026-04-15T21:04:55+05:30
Mode: explanatory

## Summary
- Small models can handle many sub-tasks in a larger AI system at dramatically lower cost and latency, but they fail on hard reasoning unless guarded by a stronger model.
- The practical pattern is: strong model decomposes and verifies; small models execute bounded, well-specified tasks; strong model checks and assembles.
- Most small-model failures are not due to model size alone — they are correctness failures (wrong output that passes simple checks) and adherence failures (wrong output shape, wrong format).
- The best current guardrail is a strong verifier that rejects outputs from the small model before the user sees them.
- This is not speculative. RouteLLM, HybridLLM, SWE-bench, and Chatbot Arena data provide quantitative measurements of when delegation works and when it fails.

## Overview
Delegating tasks to smaller LLMs is an attractive cost and latency strategy, but naive delegation often produces lower-quality results. This note investigates when small-model delegation works reliably, what kinds of failures are most common, and what guardrails the current literature supports.

The conclusion is nuanced. Small models are trustworthy for bounded, predictable sub-tasks under strong verification, but not for open-ended reasoning, multi-step synthesis, or tasks where correctness is hard to check cheaply. For practical systems, the right answer is usually a routing architecture: route easy calls to a small model, hard calls to a large model, and always run a strong verifier on outputs the user might see.

## Background
LLM providers and users want to reduce per-call costs and improve latency. Small models are faster and cheaper by orders of magnitude, but they are weaker on benchmarks, especially on reasoning-heavy or multi-step tasks. The question is whether a large model can supervise a fleet of small models to get most of the large model's quality at a fraction of the cost.

There is now enough public data (RouteLLM, HybridLLM, SWE-bench agent results, Chatbot Arena Elo) to answer this more precisely than the usual "it depends."

## Core Analysis

### Definition
"Small model" here means an LLM with ≤8B active parameters (or its quantized equivalent). "Large model" means the GPT-4 class or comparable frontier model.

### Mechanism / How delegation works

Delegation works when:
1. The task is bounded and well-specified ("convert this JSON to this schema", "tag this paragraph with categories from this list", "extract phone numbers from this text")
2. The output is easy to verify (schema, pattern-match, simple heuristics, or a fast second-pass LLM)
3. The small model does not need to plan, decompose, or synthesize across multiple steps
4. A strong verifier (rules + possibly a large model) rejects or corrects small-model mistakes before they reach the user

Delegation fails when:
1. The task requires multi-step reasoning
2. The small model's training data and reasoning patterns differ materially from what the task requires
3. Subtle errors are hard to detect with simple checks

RouteLLM provides a useful framework. It trained a router to predict whether GPT-4-turbo or Mixtral-8x7B should handle a given query. At the same quality, the router could use the small model for about 60% of calls in the Chatbot Arena distribution. The router's decisions improved when it also considered the actual assistant-level response.

HybridLLM is another routing approach that combined a local lightweight classifier with uncertainty estimates to route between a small and large model.

### Variants or categories

There are at least three distinct delegation strategies:

1. **Pure routing**: every call goes to EITHER the small model or the large model. RouteLLM and HybridLLM are examples. Safe for user-facing calls.
2. **Local with fallback**: the small model tries first. If its output does not pass a verifier, the large model retries. Works well when the small model's failure rate is low.
3. **Parallel speculative**: both models run in parallel. If the small model's output matches the large model's, return the small model's output (faster). If they differ, return the large model's. Expensive but gives small-model speed where possible without risk.

### Why quality degrades without guardrails
SWE-bench agent results show that small-model-based agents fail more often on hard issues. The main failure modes are:
- Planning errors (wrong approach to the problem)
- Failure to use tools correctly
- Producing output that looks correct but is wrong

Simple output validation ("does the JSON parse?", "is the score in range?") catches format errors but not content errors. A stronger verifier (a large model) catches more but costs more.

### Limits / common misunderstandings

The most common misunderstanding is that small models fail mainly on complex reasoning. In practice, they also fail on simple tasks where correctness is hard to verify cheaply — for example, summarization quality, subtle factual errors, or style adherence.

A second misunderstanding is that fine-tuning a small model on task-specific data eliminates the problem. It helps but does not make the small model trustworthy for high-stakes open-ended tasks. The RouteLLM authors note that small models are weaker at consistently following system-level instructions.

A third misunderstanding is that routing by topic is good enough. It is not. The RouteLLM routing analysis shows that routing quality improves significantly when the router considers the actual query content and optionally the assistant-level response, not just a topic label.

## Evidence and Sources

### Claim cluster 1: Routing can save cost without quality loss
- RouteLLM provides the strongest evidence. At parity quality with GPT-4-turbo, Mixtral-8x7B could handle about 60% of Chatbot Arena queries, reducing cost substantially.
- HybridLLM supports a similar claim with a different routing architecture.
- These results are strongest for the Chatbot Arena distribution, which skews toward open-ended chat. Results may differ for other distributions.

### Claim cluster 2: Small models fail on hard reasoning
- SWE-bench agent results consistently show smaller models failing on hard multi-step issues.
- The failure pattern is not just lower accuracy — it is qualitatively different, with more planning errors and more incorrect outputs that look superficially correct.

### Claim cluster 3: Verification is the critical guardrail
- Across the routing literature, the key to safe delegation is a strong verifier.
- Simple verification (format, schema) catches many but not all failures.
- A large-model verifier catches more but reduces cost savings.
- No single verification method catches everything.

## Uncertainties and Competing Views

High-confidence claims:
- Routing architectures can reduce cost substantially without quality loss for a meaningful fraction of real-world LLM calls.
- Small models fail on multi-step reasoning tasks in ways simple verification does not catch.
- A strong verifier is necessary for safe delegation.

Medium-confidence claims:
- The optimal router is task-distribution-dependent, and a router trained on one distribution may not generalize well.
- The long-term trend (models getting cheaper) may reduce the value of routing over time, but this is not yet clear.

Competing views:
- Some argue that models are getting cheap enough that routing is unnecessary. Others argue that as models improve, the "small model" will eventually be good enough for most tasks without routing.
- Neither view is currently supported by strong evidence.

What evidence would change the conclusion:
- Long-running production A/A tests comparing routed vs. single-model systems on user satisfaction.
- Better task taxonomies that predict small-model failure modes more precisely than current routing approaches.

## Practical Takeaways

### When to use small-model delegation
- Bounded extraction, classification, or formatting tasks
- Tasks where verification is cheap and reliable
- High-volume tasks where per-call cost or latency matters
- Tasks where the small model has demonstrably matched the large model's accuracy on your specific distribution

### When NOT to use small-model delegation
- Multi-step reasoning or planning
- Tasks where subtle errors are hard to detect
- High-stakes tasks where a single error matters
- Tasks the small model has not been systematically evaluated on

### Implementation approach
1. Start with a pure routing architecture. Log every call.
2. Audit a sample of small-model responses with the large model.
3. Measure routing accuracy, end-to-end quality, and cost.
4. Only then consider speculative or fallback architectures.
5. Always run a verifier on small-model outputs before returning to the user.

## References
1. [RouteLLM: Learning to Route LLMs with Preference Data](https://arxiv.org/abs/2406.18665) — Primary. Provides the routing framework and Chatbot Arena routing results. Strongest evidence for cost vs. quality tradeoffs.
2. [HybridLLM: Cost-Efficient Hybrid Inference](https://arxiv.org/abs/2404.13375) — Primary. Another routing approach with a different architecture.
3. [Chatbot Arena](https://chat.lmsys.org) — Primary. Source of the comparison data used in RouteLLM.
4. [SWE-bench](https://www.swebench.com/) — Primary. Source of agent-task performance data showing small-model failures on hard issues.
5. [Aristo - AI2's Reasoning Work](https://allenai.org/aristo) — Secondary. Background on multi-step reasoning failures in smaller models.
