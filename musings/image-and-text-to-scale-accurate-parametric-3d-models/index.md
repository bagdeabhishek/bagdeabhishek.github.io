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
- **Verdict: Only worth doing if you constrain the scope to mechanical/product-style objects and build a harness around strong existing models plus CAD tools.** It is **probably not worth doing** as a pure end-to-end “any image or words to perfect parametric CAD” model project today.
- The most realistic architecture is **not** a single generative model. It is a **multimodal CAD agent stack**: perception -> scale estimation -> retrieval of reference parts/templates -> CAD program synthesis -> geometry validation -> repair loop.
- For **text-only** generation, the field is moving fast and tool-using LLM systems are already competitive. For **image-to-parametric CAD**, the problem is much harder, and single-image exact recovery remains fragile unless the object family is constrained.
- If you want outputs like “Xbox controller battery cover with a GoPro mount baked in,” the best path is a **reference-model composition harness** with part retrieval and constraint-based merging, not pure text-to-geometry generation.
- Frontier models such as **Claude Opus 4.7** look promising for planning, tool use, and code generation, but the strongest public evidence still supports using them inside a **verified CAD loop**, not trusting them as one-shot CAD generators.

## Overview
This note surveys how to build a system that converts either:
1. **images with scale information**, or
2. **natural language requests**

into **scale-accurate, editable parametric 3D models**.

The target here is not generic mesh output. It is closer to **real CAD**: B-rep solids, CAD command programs, or other structured parametric representations that can be modified, measured, and manufactured.

That distinction matters. A large fraction of “text-to-3D” and “image-to-3D” progress is aimed at meshes, NeRFs, Gaussian splats, or visually plausible shapes. Those are useful for graphics, but they are not good enough when you need dimensioned, editable parts with manufacturable geometry.

## Background
There are now several overlapping research directions:
- **text-to-3D** for graphics
- **image-to-3D** for reconstruction
- **text-to-CAD** for executable parametric programs
- **image-to-CAD** for reverse engineering from photos or rendered images
- **B-rep generation** for direct CAD-native geometry
- **agentic CAD systems** where a model controls FreeCAD, CadQuery, or OpenSCAD with verification loops

The central difficulty is that parametric CAD combines:
- discrete modeling steps,
- continuous dimensions,
- topology,
- constraints,
- exact validity,
- and often references to sub-features like faces, edges, or sketch entities.

This makes CAD much less forgiving than code generation or mesh generation.

For your use case, there are three especially important distinctions:
1. **Image-to-3D is not image-to-CAD.** A good mesh prior is not the same as a good parametric model.
2. **Single-view estimation is not scale-accurate reverse engineering.** Scale needs explicit cues or calibrated measurement.
3. **Assembly/edit tasks are not object generation tasks.** “Battery cover with a GoPro mount” is better framed as template retrieval + constrained composition than as fully novel generation.

## Core Analysis
### Problem framing
You are really asking for one system with at least three subproblems:

1. **Geometry recovery**
   - infer object shape from words or images
2. **Scale recovery**
   - infer true dimensions from cues such as fiducials, known objects, camera calibration, or user measurements
3. **Parametric reconstruction or composition**
   - turn the shape into editable CAD operations, dimensions, sketches, and feature relationships

These can be solved in different ways depending on the input mode.

### Pipeline mental model
The most practical pipeline today is:
1. **Interpret the input**
   - text prompt, one or more photos, optional reference part IDs, optional known dimensions
2. **Estimate or anchor scale**
   - fiducial marker, ruler, ArUco tag, known dimension, camera intrinsics, or multi-view reconstruction
3. **Classify the task type**
   - pure generation, reverse engineering, edit existing part, or compose existing parts
4. **Retrieve priors**
   - reference CAD models, part libraries, object templates, feature libraries, mounting standards
5. **Synthesize CAD program or parametric structure**
   - CadQuery, FreeCAD Python, OpenSCAD, STEP-oriented structured representation, or patch-based CAD decoder
6. **Execute and validate**
   - compile the geometry, check solid validity, measure dimensions, compare against image/text constraints
7. **Repair or clarify**
   - ask for missing dimensions, fix invalid geometry, or refine with visual/numeric feedback

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

A simple scale anchoring equation looks like:

$$
\text{scale} = \frac{d_{real}}{d_{image}} \cdot f
$$
Plain English: the global scale can be estimated from a known real-world distance, the corresponding image measurement, and camera calibration terms.
Variables:
- $d_{real}$: known real-world distance or marker size.
- $d_{image}$: measured image-space extent for the same object.
- $f$: camera-dependent conversion term derived from calibration or pose.

The exact implementation depends on whether you use calibrated monocular geometry, multi-view reconstruction, or fiducial pose estimation.

### Method families
#### 1. Script-first text-to-CAD generation
**What it does:** maps text into executable CAD code such as CadQuery, FreeCAD Python, or OpenSCAD.

**How it helps:** gives you editable parametric outputs immediately and lets you verify geometry by execution.

**Why it exists:** LLMs already know Python reasonably well, and script-based CAD languages are much easier to target than opaque CAD kernels or raw B-rep graphs.

**What it is good at:**
- simple to moderate mechanical parts
- brackets, mounts, covers, enclosures, holders
- iterative edits through code regeneration

**What it does not solve well:**
- under-specified prompts
- exact assembly reasoning
- arbitrary complex consumer products in one shot
- precise scale without explicit dimensions

Representative work:
- Text-to-CadQuery (2025)
- CAD-Coder (2025)
- ProCAD (2026)
- CADSmith (2026)
- FutureCAD (2026)
- STEP-LLM (2026)

#### 2. Image-to-CAD through factorization
**What it does:** splits image-to-CAD into subproblems such as discrete structure prediction plus continuous parameter prediction.

**How it helps:** reduces the difficulty of directly predicting a full CAD program from pixels.

**Why it exists:** CAD outputs mix discrete and continuous structure, which is hard to learn end-to-end.

**What it is good at:**
- object families with repeated structures
- category-level reconstruction
- single-object reverse engineering where the part grammar is constrained

**What it does not solve well:**
- unconstrained arbitrary objects
- exact scale from a single casual photo
- complicated assembly composition without external priors

Representative work:
- Img2CAD (2024/2025)
- CADCrafter (CVPR 2025)
- CADDreamer (2025)

#### 3. Direct B-rep or CAD-native generation
**What it does:** generates CAD-native geometry directly, such as B-rep structures or editable surface patches.

**How it helps:** avoids lossy mesh intermediates and can produce cleaner engineering-style geometry.

**Why it exists:** meshes are insufficient for editable CAD and manufacturing workflows.

**What it is good at:**
- compact, structured geometry
- higher-fidelity CAD-like outputs
- direct export to STEP-like workflows

**What it does not solve well:**
- robust long-horizon feature logic
- exact dimension handling from vague prompts
- compositional edits unless tied to feature logic or retrieval

Representative work:
- OpenECAD (2024)
- CMT / mmABC (2025)
- GraphBrep (2025)
- AutoBrep (2025)
- DreamCAD (2026)

#### 4. Agent-aided CAD generation
**What it does:** uses an LLM or multimodal model as a planner/coder controlling a real CAD engine, then validates and repairs the result using programmatic checks.

**How it helps:** moves the burden from latent memorization to tool use, retrieval, validation, and iterative correction.

**Why it exists:** CAD errors are often easier to detect with a compiler, kernel measurements, and render checks than with a purely learned loss.

**What it is good at:**
- practical workflows today
- staying current with CAD APIs via RAG rather than retraining
- combining multiple models and tools
- integrating reference parts and assemblies

**What it does not solve well:**
- very high latency if the loop is too heavy
- tasks with weak priors and poor retrieval coverage
- tasks that need strong geometric perception from a single poor image

Representative work:
- CADSmith (2026)
- ToolCAD (2026)
- AADvark / Agent-Aided Design for Dynamic CAD Models (2026)
- FreeCAD MCP projects

### Representative methods
#### Text-to-CadQuery and CAD-Coder
These are among the clearest signals that **code-first CAD generation** is a serious path. Both target **CadQuery**, which is important because it is Pythonic, executable, and measurable. CAD-Coder adds chain-of-thought and geometric reward, while Text-to-CadQuery shows that fine-tuning larger code-capable models materially improves performance.

This is a strong argument that if your target output is parametric CAD, **choosing the right output language matters more than trying to train a generic 3D model first**.

#### ProCAD and clarification-first systems
ProCAD is especially relevant to your use case because natural language CAD requests are often incomplete. For example, “Xbox controller battery cover with a GoPro mount” leaves many questions open:
- which controller generation?
- which mount standard?
- exact attachment position?
- target wall thickness and printing constraints?

The paper’s core insight is correct: **do not hallucinate missing dimensions**. Ask or infer only where justified.

#### CADSmith-style harnesses
CADSmith is one of the strongest current signals for your question because it argues directly for a **harness over fine-tuning** in many practical settings. It uses:
- multi-agent decomposition,
- retrieval over CAD docs,
- programmatic geometric validation,
- separate judge model,
- nested correction loops.

This is very close to what you should build first.

It is also the clearest connection to your Opus comment: the paper uses a stronger **Claude Opus** judge over a **Claude Sonnet** generator to reduce confirmation bias.

#### Img2CAD, CADCrafter, and CADDreamer
These show that image-conditioned parametric CAD is becoming real, but also expose the current constraints:
- synthetic training data matters a lot
- geometric features such as depth and normals matter more than raw RGB appearance
- factorization and geometry-aware conditioning are crucial
- single-view reverse engineering is still difficult

The main lesson is not “a foundation model will solve this end-to-end.” The lesson is that **image-to-CAD needs decomposition, geometry priors, and strong validation**.

#### DreamCAD and parametric surface generation
DreamCAD is impressive because it claims multimodal CAD generation from text, images, and point clouds with editable patch-based surfaces and STEP output. But its representation is still not the same as a feature-history CAD model. It is closer to a CAD-friendly surface generator than to a fully faithful editable design history.

That means it is exciting as a geometry backbone, but it does not remove the need for a harness if you want exact product edits and compositions.

### Tradeoffs
#### End-to-end fine-tuned model
Pros:
- can become excellent for a narrow domain
- potentially lower latency at inference than a big agent loop
- can absorb repetitive priors for one object family

Cons:
- expensive data and training burden
- brittle when task format or CAD API changes
- weak on retrieval, composition, and dynamic edits unless explicitly trained
- hard to guarantee exact scale and constraints

Best fit:
- industrial domain with lots of aligned paired data
- narrow part families
- you control the output grammar and evaluator

#### Harness around strong existing models
Pros:
- fastest route to a working system
- easier to integrate reference models and standards
- easier to add measurement verification and repair
- benefits immediately from stronger frontier models like Opus 4.7
- easier to keep current as tool APIs evolve

Cons:
- more system engineering
- more moving pieces
- potentially higher latency and inference cost
- can feel less elegant than a single end-to-end model

Best fit:
- real product prototype now
- mixed workloads: text, images, edits, compositions
- when exactness matters more than demo quality

#### Hybrid path
This is the best long-term plan:
- start with a harness
- collect traces, corrections, CAD execution logs, and failure cases
- fine-tune only once you know what the harness consistently fails on

This mirrors what the best current CAD papers are converging on: code-first generation, verifier-guided repair, and post-training with geometric reward.

### Practical guidance
#### Verdict
**Verdict: Only worth doing if you constrain the domain and build a harness around existing SOTA models plus CAD tools.**

More explicitly:
- **Worth doing as a serious applied research / product prototype:** yes
- **Worth doing as a broad “any photo or words to exact CAD” startup claim:** probably not
- **Worth doing for constrained families like mounts, covers, brackets, adapters, enclosures, battery doors, fixtures:** yes
- **Worth doing if you need one-shot single-image exact reverse engineering of arbitrary objects:** no, not yet

#### Why this verdict follows from the evidence
- **Alternatives are getting strong** in text-to-CAD and tool-assisted CAD generation.
- **Image-to-CAD is improving**, but exact scale-accurate single-view reverse engineering remains fragile.
- **Delivery is feasible today** as an agentic CAD harness, especially with FreeCAD/CadQuery/OpenCascade.
- **Implementation burden is high** for end-to-end training because you need aligned datasets, CAD validation, and often synthetic data pipelines.
- **The meaningful gap still exists** in multimodal, scale-aware, compositional, reference-grounded CAD editing.

In other words: there is room for a useful system, but not for a naive one.

#### Best implementation strategies
##### Option A — Harness-first product system (recommended)
Build a multimodal pipeline with these modules:
1. **Input parser**
   - text request, photos, optional known measurements, optional existing CAD references
2. **Scale subsystem**
   - ArUco/AprilTag/ruler detection
   - camera calibration or structure-from-motion if multi-view
   - optional manual dimension confirmation UI
3. **Task router**
   - classify as: generate from text, reverse-engineer from image, edit existing CAD, compose reference parts
4. **Retrieval subsystem**
   - fetch templates from STEP/FreeCAD/CadQuery library
   - use CAD similarity search for known parts
   - fetch standards such as GoPro mount geometry, screw standards, dovetails, snap fits
5. **CAD planner/coder**
   - use frontier model (e.g. Opus 4.7 or similar) to produce a structured spec and then CAD code
6. **Kernel execution + validation**
   - OpenCascade / CadQuery / FreeCAD execution
   - validate bounding box, volume, face counts, wall thickness, mounting positions, boolean success
7. **Repair loop**
   - cheaper model or same model revises code from exact failure report
8. **Judge loop**
   - stronger model or separate critic reviews rendered views + measurements

This is the highest-probability path.

##### Option B — Reference-model composition system for edit tasks
This is the best answer to prompts like:
> create xbox controller battery cover with a gopro mount baked in

Pipeline:
1. identify object family and exact product variant
2. retrieve or reconstruct base battery-cover geometry
3. retrieve GoPro mount reference geometry and mating constraints
4. choose attach surface and orientation
5. solve attachment constraints and clearance
6. generate merged CAD feature tree
7. validate printability and fit dimensions

This is much more reliable than free-form generation because the system only needs to infer **how to combine known parts**, not invent everything from scratch.

##### Option C — Fine-tune a dedicated text-to-CAD model
Do this only if:
- you have large paired datasets,
- your output dialect is fixed,
- and your object family is narrow enough.

Good targets:
- CadQuery
- STEP-oriented structured representation
- OpenSCAD for simpler printable objects

Bad target for a first effort:
- arbitrary feature-history CAD across many product families with no retrieval layer

##### Option D — Fine-tune image-to-CAD for constrained categories
Best when all of these are true:
- objects belong to repeated families,
- training images are available with matched CAD,
- scale cues are standardized,
- and you can tolerate category-specific models.

For example:
- furniture components,
- enclosures,
- brackets,
- fixtures,
- consumer accessory shells.

#### How to use Opus 4.7 specifically
The most defensible use of **Claude Opus 4.7** is as:
- planner,
- spec normalizer,
- CAD code generator,
- or judge/refiner inside a verified harness.

Why:
- Anthropic explicitly positions it as strong for coding, agents, and vision.
- CADSmith-style results support the value of a strong judge model in a geometric loop.
- MCP-FreeCAD ecosystems already exist for Claude-driven CAD automation.

But do **not** assume “Opus 4.7 alone solves CAD.” Public evidence still favors **tool use + validation + retrieval**.

#### Recommended technical stack
For a practical v1:
- **CAD engine:** CadQuery + OpenCascade, or FreeCAD for richer feature workflows
- **Agent layer:** Opus 4.7 or another frontier coding model
- **Judge/verification:** separate model plus programmatic geometry checks
- **Storage:** CAD part/template library in STEP + native code form
- **Retrieval:** embeddings over text descriptions + graph/shape retrieval over CAD metadata
- **Vision:** object family classification, fiducial detection, optional depth/normal estimation
- **UI:** upload photos, annotate known measurement, inspect candidate geometry, approve clarified dimensions

#### Concrete build plans
##### MVP 1 — text + reference composition
Focus only on:
- text prompt
- reference part retrieval
- GoPro mount / screw / bracket standards
- CadQuery generation with validation

This avoids the hard image perception problem and gets you to value quickly.

##### MVP 2 — photo-guided constrained reverse engineering
Add:
- one object per scene
- required fiducial marker or ruler in frame
- constrained part families only
- optional 2–4 views instead of single image

This is much more realistic than “upload one random phone photo and get exact CAD.”

##### MVP 3 — assembly-aware editing harness
Support:
- load existing STEP or reference CAD
- select region/part to modify
- attach or replace standard subcomponents
- validate fit and clearances

This is likely the highest-value business workflow.

### Open questions
- How much can retrieval and reference composition reduce the need for image-to-CAD training?
- For consumer accessories, is single-view enough if the base part is already known and only the modification is novel?
- Is STEP generation ultimately more useful than CadQuery generation for downstream manufacturing handoff?
- How much human clarification should be built into the loop before users perceive it as too interactive?

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
- Paper claims are often benchmarked on synthetic or category-constrained datasets, so generalization to messy real photos is uncertain.
- Patch-based CAD outputs and direct B-rep outputs are promising, but they are not always equivalent to clean human-authored feature histories.
- Public benchmark data for Opus 4.7 on CAD-specific tasks is still limited; much evidence is indirect through coding/agentic strength.
- Single-view reverse engineering from casual images may still require too much hidden prior knowledge for robust scale-accurate outputs.
- For many applications, retrieval + edit of existing CAD may beat generation in both accuracy and product usefulness.

## Practical Takeaways
- If you want a real system soon, **build a harness, not a pure model**.
- If you want scale accuracy from photos, **require scale cues** and preferably **multiple views**.
- If your task is “modify an existing part with a standard subcomponent,” use **retrieval + composition**, not full generation.
- Use **Opus 4.7 as a planner/coder/judge inside a CAD loop**, not as the whole solution.
- Fine-tune only after you have execution traces, validators, and a clear failure distribution.
- Constrain the domain aggressively for the first version.

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
