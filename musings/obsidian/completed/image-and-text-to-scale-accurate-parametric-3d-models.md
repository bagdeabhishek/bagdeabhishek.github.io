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
1. **Geometry recovery** - infer object shape from words or images
2. **Scale recovery** - infer true dimensions from cues such as fiducials, known objects, camera calibration, or user measurements
3. **Parametric reconstruction or composition** - turn the shape into editable CAD operations

### Pipeline mental model
The most practical pipeline today is:
1. Interpret the input - text prompt, one or more photos, optional reference part IDs, optional known dimensions
2. Estimate or anchor scale - fiducial marker, ruler, ArUco tag, known dimension, camera intrinsics, or multi-view reconstruction
3. Classify the task type - pure generation, reverse engineering, edit existing part, or compose existing parts
4. Retrieve priors - reference CAD models, part libraries, object templates, feature libraries, mounting standards
5. Synthesize CAD program or parametric structure - CadQuery, FreeCAD Python, OpenSCAD, or STEP-oriented structured representation
6. Execute and validate - compile the geometry, check solid validity, measure dimensions, compare against image/text constraints
7. Repair or clarify - ask for missing dimensions, fix invalid geometry, or refine with visual/numeric feedback

A simple scale anchoring equation:

$$
\text{scale} = \frac{d_{real}}{d_{image}} \cdot f
$$
Plain English: the global scale can be estimated from a known real-world distance, the corresponding image measurement, and camera calibration terms.
Variables:
- $d_{real}$: known real-world distance or marker size.
- $d_{image}$: measured image-space extent for the same object.
- $f$: camera-dependent conversion term derived from calibration or pose.

### Method families
#### 1. Script-first text-to-CAD generation
**What it does:** maps text into executable CAD code such as CadQuery, FreeCAD Python, or OpenSCAD.
**How it helps:** gives you editable parametric outputs immediately and lets you verify geometry by execution.
**Why it exists:** LLMs already know Python reasonably well, and script-based CAD languages are much easier to target than opaque CAD kernels.
**What it is good at:** simple to moderate mechanical parts, brackets, mounts, covers, enclosures, holders, iterative edits through code regeneration.
**What it does not solve well:** under-specified prompts, exact assembly reasoning, arbitrary complex consumer products in one shot, precise scale without explicit dimensions.
Representative work: Text-to-CadQuery (2025), CAD-Coder (2025), ProCAD (2026), CADSmith (2026), FutureCAD (2026), STEP-LLM (2026)

#### 2. Image-to-CAD through factorization
**What it does:** splits image-to-CAD into subproblems such as discrete structure prediction plus continuous parameter prediction.
**How it helps:** reduces the difficulty of directly predicting a full CAD program from pixels.
**Why it exists:** CAD outputs mix discrete and continuous structure, which is hard to learn end-to-end.
**What it is good at:** object families with repeated structures, category-level reconstruction, single-object reverse engineering where the part grammar is constrained.
**What it does not solve well:** unconstrained arbitrary objects, exact scale from a single casual photo, complicated assembly composition without external priors.
Representative work: Img2CAD (2024/2025), CADCrafter (CVPR 2025), CADDreamer (2025)

#### 3. Direct B-rep or CAD-native generation
**What it does:** generates CAD-native geometry directly, such as B-rep structures or editable surface patches.
**How it helps:** avoids lossy mesh intermediates and can produce cleaner engineering-style geometry.
**Why it exists:** meshes are insufficient for editable CAD and manufacturing workflows.
**What it is good at:** compact, structured geometry, higher-fidelity CAD-like outputs, direct export to STEP-like workflows.
**What it does not solve well:** robust long-horizon feature logic, exact dimension handling from vague prompts, compositional edits unless tied to feature logic or retrieval.
Representative work: OpenECAD (2024), CMT / mmABC (2025), GraphBrep (2025), AutoBrep (2025), DreamCAD (2026)

#### 4. Agent-aided CAD generation
**What it does:** uses an LLM or multimodal model as a planner/coder controlling a real CAD engine, then validates and repairs the result using programmatic checks.
**How it helps:** moves the burden from latent memorization to tool use, retrieval, validation, and iterative correction.
**Why it exists:** CAD errors are often easier to detect with a compiler, kernel measurements, and render checks than with a purely learned loss.
**What it is good at:** practical workflows today, staying current with CAD APIs via RAG rather than retraining, combining multiple models and tools, integrating reference parts and assemblies.
**What it does not solve well:** very high latency if the loop is too heavy, tasks with weak priors and poor retrieval coverage, tasks that need strong geometric perception from a single poor image.
Representative work: CADSmith (2026), ToolCAD (2026), AADvark / Agent-Aided Design for Dynamic CAD Models (2026), FreeCAD MCP projects

### Tradeoffs
#### End-to-end fine-tuned model
Pros: can become excellent for a narrow domain, potentially lower latency at inference than a big agent loop.
Cons: expensive data and training burden, brittle when task format or CAD API changes, weak on retrieval/composition/dynamic edits unless explicitly trained, hard to guarantee exact scale and constraints.
Best fit: industrial domain with lots of aligned paired data, narrow part families, you control the output grammar and evaluator.

#### Harness around strong existing models
Pros: fastest route to a working system, easier to integrate reference models and standards, easier to add measurement verification and repair, benefits immediately from stronger frontier models, easier to keep current as tool APIs evolve.
Cons: more system engineering, more moving pieces, potentially higher latency and inference cost.
Best fit: real product prototype now, mixed workloads, when exactness matters more than demo quality.

#### Hybrid path
Start with a harness, collect traces/corrections/CAD execution logs/failure cases, fine-tune only once you know what the harness consistently fails on.

### Practical guidance
#### Verdict
**Verdict: Only worth doing if you constrain the domain and build a harness around existing SOTA models plus CAD tools.**
- **Worth doing as a serious applied research / product prototype:** yes
- **Worth doing as a broad "any photo or words to exact CAD" startup claim:** probably not
- **Worth doing for constrained families like mounts, covers, brackets, adapters, enclosures, battery doors, fixtures:** yes
- **Worth doing if you need one-shot single-image exact reverse engineering of arbitrary objects:** no, not yet

#### Recommended technical stack
For a practical v1:
- **CAD engine:** CadQuery + OpenCascade, or FreeCAD for richer feature workflows
- **Agent layer:** Opus 4.7 or another frontier coding model
- **Judge/verification:** separate model plus programmatic geometry checks
- **Storage:** CAD part/template library in STEP + native code form
- **Retrieval:** embeddings over text descriptions + graph/shape retrieval over CAD metadata
- **Vision:** object family classification, fiducial detection, optional depth/normal estimation
- **UI:** upload photos, annotate known measurement, inspect candidate geometry, approve clarified dimensions

### Open questions
- How much can retrieval and reference composition reduce the need for image-to-CAD training?
- For consumer accessories, is single-view enough if the base part is already known and only the modification is novel?
- Is STEP generation ultimately more useful than CadQuery generation for downstream manufacturing handoff?
- How much human clarification should be built into the loop before users perceive it as too interactive?

## Evidence and Sources
### Core text-to-CAD / agentic sources
- Text-to-CadQuery (2025), CAD-Coder (2025), CADSmith (2026), ProCAD (2026), ToolCAD (2026), FutureCAD (2026), STEP-LLM (2026)
### Image-to-CAD / CAD-native geometry sources
- OpenECAD (2024), Img2CAD (2024/2025), CADCrafter (CVPR 2025), CADDreamer (2025), DreamCAD (2026), CMT / mmABC (2025)
### Practical systems and products
- CADAM, Text-to-.step, FreeCAD MCP projects, CadQueryEval, Anthropic Claude Opus 4.7 page

## Uncertainties and Competing Views
- Paper claims are often benchmarked on synthetic or category-constrained datasets.
- Patch-based CAD outputs and direct B-rep outputs are promising, but not always equivalent to clean human-authored feature histories.
- Single-view reverse engineering from casual images may still require too much hidden prior knowledge.

## Practical Takeaways
- If you want a real system soon, build a harness, not a pure model.
- If you want scale accuracy from photos, require scale cues and preferably multiple views.
- If your task is "modify an existing part with a standard subcomponent," use retrieval + composition, not full generation.
- Use Opus 4.7 as a planner/coder/judge inside a CAD loop, not as the whole solution.
- Fine-tune only after you have execution traces, validators, and a clear failure distribution.

## References
1. [Claude Opus 4.7](https://anthropic.com/claude/opus)
2. [CadQueryEval](https://danwahl.net/cadqueryeval/)
3. [OpenECAD](https://arxiv.org/abs/2406.09913)
4. [Img2CAD](https://arxiv.org/abs/2408.01437)
5. [Text2CAD](https://arxiv.org/abs/2411.06206)
6. [CADDreamer](https://arxiv.org/abs/2502.20732)
7. [Text-to-CadQuery](https://arxiv.org/abs/2505.06507)
8. [CAD-Coder](https://arxiv.org/abs/2505.19713)
9. [CADCrafter](https://openaccess.thecvf.com/content/CVPR2025/papers/Chen_CADCrafter_Generating_Computer-Aided_Design_Models_from_Unconstrained_Images_CVPR_2025_paper.pdf)
10. [DreamCAD](https://sadilkhan.github.io/dreamcad2026/)
11. [GraphBrep](https://arxiv.org/abs/2507.04765)
12. [CMT](https://arxiv.org/abs/2504.20830)
13. [PLLM](https://arxiv.org/abs/2602.12561)
14. [ProCAD](https://arxiv.org/abs/2602.03045)
15. [FutureCAD](https://arxiv.org/abs/2603.11831)
16. [CADSmith](https://arxiv.org/abs/2603.26512)
17. [TOOLCAD](https://arxiv.org/abs/2604.07960)
18. [Agent-Aided Design](https://arxiv.org/abs/2604.15184)
19. [CADAM](https://github.com/Adam-CAD/CADAM)
20. [Text-to-.step](https://github.com/Sjs2332/Text-to-.step)
21. [FreeCAD MCP](https://github.com/contextform/freecad-mcp)
22. [MCP-FreeCAD Integration](https://github.com/jango-blockchained/mcp-freecad)
