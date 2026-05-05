---
layout: default
title: Image and Text to Scale-Accurate Parametric 3D Models
permalink: /musings/image-and-text-to-scale-accurate-parametric-3d-models/
---

# Image and Text to Scale-Accurate Parametric 3D Models
_AI-assisted research synthesis. Verify critical claims with primary sources._
Status: Completed
Last updated: 2026-04-19T18:48:28+05:30
Mode: method-survey

## Summary
- **Verdict: Only worth doing if you constrain the scope to mechanical/product-style objects and build a harness around strong existing models plus CAD tools.** It is **probably not worth doing** as a pure end-to-end "any image or words to perfect parametric CAD" model project today.
- The most realistic architecture is **not** a single generative model. It is a **multimodal CAD agent stack**: perception -> scale estimation -> retrieval of reference parts/templates -> CAD program synthesis -> geometry validation -> repair loop.
- For **text-only** generation, the field is moving fast and tool-using LLM systems are already competitive. For **image-to-parametric CAD**, the problem is much harder, and single-image exact recovery remains fragile unless the object family is constrained.
- If you want outputs like "Xbox controller battery cover with a GoPro mount baked in," the best path is a **reference-model composition harness** with part retrieval and constraint-based merging, not pure text-to-geometry generation.
- Frontier models such as **Claude Opus 4.7** look promising for planning, tool use, and code generation, but the strongest public evidence still supports using them inside a **verified CAD loop**, not trusting them as one-shot CAD generators.

## Overview
This note surveys how to build a system that converts either:
1. **images with scale information**, or
2. **natural language requests**

into **scale-accurate, editable parametric 3D models**.

The target here is not generic mesh output. It is closer to **real CAD**: B-rep solids, CAD command programs, or other structured parametric representations that can be modified, measured, and manufactured.

That distinction matters. A large fraction of "text-to-3D" and "image-to-3D" progress is aimed at meshes, NeRFs, Gaussian splats, or visually plausible shapes. Those are useful for graphics, but they are not good enough when you need dimensioned, editable parts with manufacturable geometry.

## Background
There are now several overlapping research directions:
- **text-to-3D** for graphics
- **image-to-3D** for reconstruction
- **text-to-CAD** for executable parametric programs
- **image-to-CAD** for reverse engineering from photos or rendered images
- **B-rep generation** for direct CAD-native geometry
- **agentic CAD systems** where a model controls FreeCAD, CadQuery, or OpenSCAD with verification loops

The central difficulty is that parametric CAD combines discrete modeling steps, continuous dimensions, topology, constraints, exact validity, and often references to sub-features like faces, edges, or sketch entities. This makes CAD much less forgiving than code generation or mesh generation.

## Core Analysis
### Problem framing
You are really asking for one system with at least three subproblems:
1. **Geometry recovery** — infer object shape from words or images
2. **Scale recovery** — infer true dimensions from cues such as fiducials, known objects, camera calibration, or user measurements
3. **Parametric reconstruction or composition** — turn the shape into editable CAD operations, dimensions, sketches, and feature relationships

### Pipeline mental model
The most practical pipeline today is:
1. Interpret the input
2. Estimate or anchor scale
3. Classify the task type (pure generation, reverse engineering, edit existing, or compose existing)
4. Retrieve priors (reference CAD models, part libraries, object templates)
5. Synthesize CAD program or parametric structure
6. Execute and validate
7. Repair or clarify

This can be summarized as:

$$
\hat{P} = \operatorname*{argmax}_P \; \text{Fit}(P, X, T, S, R) \quad \text{s.t. valid}(P)=1
$$
Plain English: choose the CAD program or parametric model $P$ that best fits the images, text, scale cues, and retrieved references, while remaining geometrically valid.
Variables:
- $P$: candidate CAD program or parametric model.
- $X$: visual observations such as one or more photos.
- $T$: text instructions or prompt.
- $S$: scale information such as fiducials, ruler, or known dimensions.
- $R$: retrieved templates or reference parts.
- $\text{Fit}(\cdot)$: score measuring how well the output matches the input conditions.
- $valid(P)$: CAD validity constraint, such as executable code and valid solid geometry.

### Method families
#### 1. Script-first text-to-CAD generation
Maps text into executable CAD code such as CadQuery, FreeCAD Python, or OpenSCAD. Strong for simple to moderate mechanical parts (brackets, mounts, covers). Representative work: Text-to-CadQuery, CAD-Coder, ProCAD, CADSmith, FutureCAD, STEP-LLM.

#### 2. Image-to-CAD through factorization
Splits image-to-CAD into subproblems: discrete structure prediction + continuous parameter prediction. Strong for object families with repeated structures. Representative work: Img2CAD, CADCrafter, CADDreamer.

#### 3. Direct B-rep or CAD-native generation
Generates CAD-native geometry directly as B-rep structures or editable surface patches. Avoids lossy mesh intermediates. Representative work: OpenECAD, CMT/mmABC, GraphBrep, DreamCAD.

#### 4. Agent-aided CAD generation
Uses an LLM or multimodal model as a planner/coder controlling a real CAD engine, then validates and repairs the result using programmatic checks. Strongest practical path today. Representative work: CADSmith, ToolCAD, FreeCAD MCP projects.

### Representative methods
#### Text-to-CadQuery and CAD-Coder
These target CadQuery — Pythonic, executable, measurable. CAD-Coder adds chain-of-thought and geometric reward. The main insight: choosing the right output language matters more than training a generic 3D model first.

#### ProCAD and clarification-first systems
Natural language CAD requests are often incomplete. The core insight: do not hallucinate missing dimensions. Ask or infer only where justified.

#### CADSmith-style harnesses
Uses multi-agent decomposition, retrieval over CAD docs, programmatic geometric validation, separate judge model, and nested correction loops. This is very close to what should be built first.

#### Img2CAD, CADCrafter, and CADDreamer
Image-conditioned parametric CAD is becoming real, but still constrained: synthetic training data matters a lot, geometric features matter more than raw RGB, and single-view reverse engineering is still difficult.

### Tradeoffs
#### End-to-end fine-tuned model
Pros: excellent for narrow domains, lower latency. Cons: expensive data and training burden, brittle when CAD API changes, weak on retrieval and composition. Best fit: industrial domain with lots of aligned paired data.

#### Harness around strong existing models
Pros: fastest route to working system, easier to integrate reference models, benefits immediately from stronger frontier models. Cons: more system engineering, potentially higher latency. Best fit: real product prototype now.

#### Hybrid path (recommended)
Start with a harness, collect traces/corrections/failure cases, fine-tune only once you know what the harness consistently fails on.

### Practical guidance
#### Verdict
**Verdict: Only worth doing if you constrain the domain and build a harness around existing SOTA models plus CAD tools.**

#### Why this verdict follows from the evidence
- Alternatives are getting strong in text-to-CAD and tool-assisted CAD generation.
- Image-to-CAD is improving, but exact scale-accurate single-view reverse engineering remains fragile.
- Delivery is feasible today as an agentic CAD harness.
- Implementation burden is high for end-to-end training.

#### Recommended technical stack for v1
- CAD engine: CadQuery + OpenCascade, or FreeCAD
- Agent layer: frontier coding model (e.g. Opus 4.7)
- Judge/verification: separate model plus programmatic geometry checks
- Storage: CAD part/template library in STEP
- Retrieval: embeddings over text descriptions + shape retrieval
- Vision: object family classification, fiducial detection

### Open questions
- How much can retrieval and reference composition reduce the need for image-to-CAD training?
- For consumer accessories, is single-view enough if the base part is already known?
- Is STEP generation ultimately more useful than CadQuery generation for manufacturing handoff?

## Evidence and Sources
### Core text-to-CAD / agentic sources
- Text-to-CadQuery (2025) — direct CadQuery generation from text using LLM fine-tuning.
- CAD-Coder (2025) — chain-of-thought plus geometric reward for text-to-CAD.
- CADSmith (2026) — multi-agent CAD generation with programmatic geometric validation.
- ProCAD (2026) — proactive clarification before CAD synthesis.
- ToolCAD (2026) — tool-using LLM agents trained for CAD.
- FutureCAD (2026) — LLM-driven program generation with B-rep primitive grounding.
- STEP-LLM (2026) — direct generation of STEP models from language.

### Image-to-CAD / CAD-native geometry sources
- OpenECAD (2024) — editable CAD generation from images with VLM fine-tuning.
- Img2CAD (2024/2025) — VLM-assisted conditional factorization from images to CAD.
- CADCrafter (CVPR 2025) — latent diffusion for image-to-parametric CAD from unconstrained images.
- CADDreamer (2025) — CAD B-rep generation from single-view images.
- DreamCAD (2026) — multimodal CAD generation with editable patch-based surfaces.
- CMT / mmABC (2025) — multimodal conditional B-rep generation with large multimodal dataset.

### Practical systems and products
- CADAM — open-source browser-based text/image-to-CAD app using OpenSCAD.
- Text-to-.step — practical FreeCAD-based text-to-B-rep system.
- FreeCAD MCP projects — practical Claude/LLM-to-FreeCAD control bridges.
- CadQueryEval — public benchmark for natural-language-to-CadQuery generation.
- Anthropic Claude Opus 4.7 page — evidence for current frontier coding/vision/agent positioning.

## Uncertainties and Competing Views
- Paper claims are often benchmarked on synthetic or category-constrained datasets.
- Patch-based CAD outputs and direct B-rep outputs are not always equivalent to clean human-authored feature histories.
- Single-view reverse engineering from casual images may still require too much hidden prior knowledge.
- For many applications, retrieval + edit of existing CAD may beat generation in both accuracy and product usefulness.

## Practical Takeaways
- If you want a real system soon, **build a harness, not a pure model**.
- If you want scale accuracy from photos, **require scale cues** and preferably **multiple views**.
- If your task is "modify an existing part with a standard subcomponent," use **retrieval + composition**, not full generation.
- Use **Opus 4.7 as a planner/coder/judge inside a CAD loop**, not as the whole solution.
- Fine-tune only after you have execution traces, validators, and a clear failure distribution.

## References
1. [Claude Opus 4.7](https://anthropic.com/claude/opus) — current frontier model positioning across coding, vision, and agents.
2. [CadQueryEval](https://danwahl.net/cadqueryeval/) — public benchmark for natural-language-to-CadQuery generation.
3. [OpenECAD: An Efficient Visual Language Model for Editable 3D-CAD Design](https://arxiv.org/abs/2406.09913) — image-conditioned editable CAD generation with VLM fine-tuning.
4. [Img2CAD: Reverse Engineering 3D CAD Models from Images through VLM-Assisted Conditional Factorization](https://arxiv.org/abs/2408.01437) — factorized image-to-CAD.
5. [Text2CAD: Text to 3D CAD Generation via Technical Drawings](https://arxiv.org/abs/2411.06206) — text to technical drawings to CAD reconstruction.
6. [CADDreamer: CAD Object Generation from Single-view Images](https://arxiv.org/abs/2502.20732) — single-view image to CAD B-rep.
7. [Text-to-CadQuery: A New Paradigm for CAD Generation with Scalable Large Model Capabilities](https://arxiv.org/abs/2505.06507) — fine-tuned LLMs for CadQuery generation.
8. [CAD-Coder: Text-to-CAD Generation with Chain-of-Thought and Geometric Reward](https://arxiv.org/abs/2505.19713) — CoT + GRPO for text-to-CAD.
9. [CADCrafter: Generating Computer-Aided Design Models from Unconstrained Images](https://openaccess.thecvf.com/content/CVPR2025/papers/Chen_CADCrafter_Generating_Computer-Aided_Design_Models_from_Unconstrained_Images_CVPR_2025_paper.pdf) — image-to-parametric CAD from unconstrained images.
10. [DreamCAD: Scaling Multi-modal CAD Generation using Differentiable Parametric Surfaces](https://sadilkhan.github.io/dreamcad2026/) — multimodal CAD generation from text, images, and point clouds.
11. [GraphBrep: Learning B-Rep in Graph Structure for Efficient CAD Generation](https://arxiv.org/abs/2507.04765) — graph-based B-rep generation.
12. [CMT: A Cascade MAR with Topology Predictor for Multimodal Conditional CAD Generation](https://arxiv.org/abs/2504.20830) — multimodal conditional B-rep generation on mmABC.
13. [PLLM: Pseudo-Labeling Large Language Models for CAD Program Synthesis](https://arxiv.org/abs/2602.12561) — self-training CAD program synthesis.
14. [Clarify Before You Draw: Proactive Agents for Robust Text-to-CAD Generation](https://arxiv.org/abs/2602.03045) — clarification-first CAD agents.
15. [Towards High-Fidelity CAD Generation via LLM-Driven Program Generation and Text-Based B-Rep Primitive Grounding](https://arxiv.org/abs/2603.11831) — FutureCAD and B-rep grounding.
16. [CADSmith: Multi-Agent CAD Generation with Programmatic Geometric Validation](https://arxiv.org/abs/2603.26512) — multi-agent validated CAD generation.
17. [TOOLCAD: Exploring Tool-Using Large Language Models in Text-to-CAD Generation with Reinforcement Learning](https://arxiv.org/abs/2604.07960) — RL-trained CAD tool-use agents.
18. [Agent-Aided Design for Dynamic CAD Models](https://arxiv.org/abs/2604.15184) — assembly-aware agentic CAD with constraints.
19. [CADAM](https://github.com/Adam-CAD/CADAM) — open-source text/image-to-CAD web application.
20. [Text-to-.step](https://github.com/Sjs2332/Text-to-.step) — practical FreeCAD-based text-to-STEP system.
21. [FreeCAD MCP](https://github.com/contextform/freecad-mcp) — Claude-to-FreeCAD integration.
22. [MCP-FreeCAD Integration](https://github.com/jango-blockchained/mcp-freecad) — AI assistant integration with FreeCAD via MCP.
