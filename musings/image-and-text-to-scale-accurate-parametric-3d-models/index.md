---
layout: default
title: Image and Text to Scale-Accurate Parametric 3D Models
permalink: /musings/image-and-text-to-scale-accurate-parametric-3d-models/
---

# Image and Text to Scale-Accurate Parametric 3D Models
_AI-assisted research synthesis. Verify critical claims with primary sources._
Status: Completed
Last updated: 2026-04-18T14:38:47+05:30
Mode: method-survey

## Summary
- **Verdict: Only worth doing if you constrain the scope to mechanical/product-style parts.** Full-scene reconstruction from images alone is still a research-grade problem.
- The best pipeline today: image → 3D reconstruction (point cloud/mesh) → parametric fitting (B-rep/CSG) → scale calibration from reference objects or metadata.
- If you want outputs like “Xbox controller battery cover with a GoPro mount back” style parts, programmatic CAD generation from text is viable now. If you want photorealistic whole-object reconstruction from a single phone photo with accurate dimensions, the tech isn't there yet.
- Smartphone photogrammetry is good enough for basic dimensional extraction when you have scale references, but precision degrades for complex geometries and reflective surfaces.
- The strongest recent systems combine multi-view reconstruction with learned shape priors; purely single-image approaches still produce dimensionally ambiguous outputs.
- Scale accuracy is the hardest sub-problem. Most reconstruction pipelines produce scale-ambiguous outputs. Adding scale requires either known reference objects, depth sensors, or metadata-driven calibration.

## Overview
This note surveys the problem of converting images (photos, screenshots, sketches) and text descriptions into scale-accurate parametric 3D models suitable for manufacturing, CAD, and 3D printing. The core challenge is that photographs are 2D projections with unknown scale, camera parameters, and occlusions — recovering a dimensionally accurate 3D model requires solving an underdetermined inverse problem.

The note focuses on three sub-problems:
1. Image-to-3D reconstruction (geometry extraction from photos)
2. Parametric CAD generation (producing editable B-rep/CSG/STEP models, not just meshes)
3. Scale calibration (making the output dimensionally accurate)

## Background
Traditional 3D reconstruction (photogrammetry, structure from motion) can produce accurate meshes from multiple calibrated images. But these meshes are not parametric CAD models — they can't be easily edited in Fusion 360 or SolidWorks, and they lack the manufacturing semantics (holes, fillets, extrusions) that engineers need.

The recent explosion in AI-driven 3D generation (NeRF, Gaussian Splatting, diffusion-based 3D) has dramatically improved reconstruction quality from sparse views, but most systems still output meshes or point clouds, not parametric models. A parallel research thread on programmatic CAD generation (text-to-CAD, image-to-CAD) has emerged, aiming to produce editable CAD formats directly.

## Core Analysis

### Problem decomposition

The overall problem decomposes into three coupled sub-problems:

1. **Geometry extraction**: Recover 3D shape from 2D observations. This is the most-studied sub-problem and the one where AI has made the most progress.

2. **Parametric fitting**: Convert raw geometry (mesh/point cloud) into parametric CAD primitives (extrusions, revolves, sweeps, Boolean combinations). This is the "reverse engineering" problem in CAD and remains challenging.

3. **Scale calibration**: Determine absolute dimensions from scale-ambiguous reconstruction. Requires either known reference objects, camera metadata (focal length, sensor size), depth sensor data, or user-provided measurements.

### Why scale accuracy is the hardest part

Most 3D reconstruction pipelines produce outputs that are correct up to an unknown scale factor. A reconstruction might capture the proportions perfectly but be 2x too large or small. For visualization (games, VR, film), this doesn't matter. For manufacturing and CAD, it's a dealbreaker. A 3D-printed part that's 10% off in any dimension is scrap.

Scale calibration methods:
- **Reference object**: Include a ruler, coin, or known-size fiducial in the photo
- **Depth sensor**: Use LiDAR or structured light (iPhone Pro, Intel RealSense)
- **Camera metadata**: Use EXIF data (focal length, sensor size) with geometric constraints
- **Multi-view triangulation**: With enough views and known camera motion, scale can be recovered
- **User annotation**: Ask the user to measure and input one known dimension

### Pipeline families

#### Pipeline A: Image → Mesh → Parametric CAD

1. Capture multiple photos or a video of the object
2. Run photogrammetry (COLMAP, RealityCapture, Polycam) to get a dense mesh
3. Segment the mesh into geometric primitives
4. Fit parametric surfaces (planes, cylinders, cones) to segments
5. Reconstruct the CAD feature tree (extrusions, cuts, fillets)
6. Calibrate scale from reference

**Strengths**: Can handle complex organic shapes. Mature photogrammetry tooling.
**Weaknesses**: Parametric fitting step is brittle. Feature tree reconstruction is an active research problem. Scale calibration requires extra steps.

#### Pipeline B: Image → Direct CAD Generation

1. Input one or more images of the part
2. Use an AI model trained to output CAD construction sequences (sketch → extrude → cut → fillet sequences)
3. The model predicts both geometry and dimensions
4. Post-process to ensure valid CAD (constraint solving)

**Strengths**: Outputs are natively editable. Growing research area with improving results.
**Weaknesses**: Current models are limited to relatively simple parts. Training data is scarce. Scale accuracy depends on training distribution.

#### Pipeline C: Text → Programmatic CAD

1. User describes the part in text (dimensions, features, constraints)
2. LLM generates CAD code (OpenSCAD, CadQuery, Build123d, FreeCAD Python)
3. Render and iterate with user feedback

**Strengths**: Natively parametric and editable. Good for mechanical/structural parts with clear specifications. Viable today for many use cases.
**Weaknesses**: Requires the user to specify dimensions. Can't handle "make it look like this photo" style inputs. LLM-generated CAD code can have geometric errors.

### Current state of the art

**Best for reconstruction quality**: Multi-view photogrammetry (COLMAP + MVS) or Neural Radiance Fields (3D Gaussian Splatting) for visual quality.

**Best for parametric output from images**: Emerging research (B-rep reconstruction from point clouds, sketch-and-extrude prediction from multi-view). Not production-ready for complex parts.

**Best for parametric output from text**: LLM-driven CAD code generation (FreeCAD MCP integration, text-to-STEP systems). Viable now for mechanical parts with clear specifications.

**Best for scale accuracy**: Reference-object photogrammetry with calibrated cameras. Adding a scale bar or known-size object is the most reliable method.

## Method Survey

### 1. Classical photogrammetry (COLMAP + MVS)
- **What it does**: Multi-view stereo reconstruction from images. Produces dense point clouds and meshes.
- **Scale accuracy**: Scale-ambiguous without reference or known camera parameters.
- **Parametric output**: No — outputs meshes only.
- **Maturity**: Production-ready. Used in film, surveying, cultural heritage.
- **References**: COLMAP (Schönberger & Frahm, 2016), OpenMVS.

### 2. Neural Radiance Fields (NeRF, 3D Gaussian Splatting)
- **What it does**: Neural scene representation from sparse views. Produces photorealistic novel views and can extract meshes.
- **Scale accuracy**: Scale-ambiguous. Some variants (BARF, SCNeRF) can recover camera parameters including scale.
- **Parametric output**: No — outputs radiance fields or extracted meshes.
- **Maturity**: Research-grade to early production. Visual quality excellent; geometric accuracy improving.
- **References**: Mildenhall et al. (2020), Kerbl et al. (2023).

### 3. Single-image 3D reconstruction (Zero-1-to-3, Stable Zero123, TripoSR)
- **What it does**: Generate 3D mesh from a single image using diffusion priors.
- **Scale accuracy**: Scale-ambiguous. Output dimensions are arbitrary.
- **Parametric output**: No — outputs meshes.
- **Maturity**: Early product stage. Quality varies widely by object type.
- **References**: Zero-1-to-3 (Liu et al., 2023), TripoSR (Stability AI, 2024).

### 4. Smartphone photogrammetry apps (Polycam, KIRI Engine, RealityScan)
- **What it does**: User-facing apps that capture video and produce 3D models.
- **Scale accuracy**: Scale-ambiguous unless using LiDAR (iPhone Pro models). Even with LiDAR, absolute accuracy is ~1-2 cm.
- **Parametric output**: No — outputs meshes. Some offer basic measurement tools.
- **Maturity**: Production. Good enough for visualization, not for precision manufacturing.

### 5. B-rep reconstruction from point clouds (Inverse CAD)
- **What it does**: Fit parametric surfaces and feature trees to point cloud/mesh data.
- **Scale accuracy**: Inherits scale from input reconstruction. If input is calibrated, output is calibrated.
- **Parametric output**: Yes — produces B-rep CAD models.
- **Maturity**: Research to early commercial. Works for simple parts with clear planar/cylindrical surfaces.
- **References**: Point2CAD (2024), B-rep reverse engineering literature.

### 6. Text-to-CAD / Image-to-CAD generation
- **What it does**: AI model directly outputs CAD construction sequences (sketches + extrusions + Boolean operations) from text or image input.
- **Scale accuracy**: Varies. Text-to-CAD systems often require explicit dimension input. Image-to-CAD systems may predict dimensions from visual cues.
- **Parametric output**: Yes — natively parametric and editable.
- **Maturity**: Research. Rapidly improving but limited part complexity.
- **References**: Text2CAD (2023), CAD2Sketch (2024), mmABC dataset, FutureCAD, TOOLCAD, CADSmith, CADAM.

### 7. LLM-driven programmatic CAD
- **What it does**: LLM writes CAD code (Python, OpenSCAD) from natural language descriptions.
- **Scale accuracy**: User specifies dimensions in text. As accurate as the user's specifications.
- **Parametric output**: Yes — code-based CAD is inherently parametric.
- **Maturity**: Viable for simple-to-moderate parts. Active open-source ecosystem.
- **References**: FreeCAD MCP integration, Text-to-.step, CadQuery + LLM workflows.

## Evidence and Sources

### Reconstruction quality benchmarks
- COLMAP + MVS achieves sub-millimeter accuracy with proper calibration and sufficient views.
- 3D Gaussian Splatting achieves superior visual quality but comparable or slightly worse geometric accuracy vs. classical MVS.
- Single-image methods (Zero-1-to-3, TripoSR) achieve visually plausible but geometrically unreliable results.

### Scale accuracy evidence
- iPhone LiDAR: ~1 cm accuracy at close range (1-2 m), degrading with distance.
- Reference object photogrammetry: ~0.1-0.5 mm accuracy with proper setup (known-size calibration target, controlled lighting).
- Consumer photogrammetry without reference: scale error of 5-20% is typical.

### Parametric CAD from 3D data
- Point2CAD (2024): Reconstructs CAD feature trees from point clouds. Works well for prismatic parts with planar and cylindrical features.
- B-rep generation from images (2025-2026): mmABC, FutureCAD, and related works show rapid progress but are still limited to parts with <20 features.

### Programmatic CAD from text
- FreeCAD MCP integration: Demonstrated viable for mechanical parts like brackets, adapters, enclosures.
- Text-to-.step: Practical system for generating STEP files from text descriptions using FreeCAD.
- CADSmith: Multi-agent CAD generation with geometric validation. Improves output correctness.

## Practical Guidance

### When to use each approach

| Use case | Recommended approach |
|----------|---------------------|
| Reverse-engineer a simple mechanical part | Photogrammetry + reference object + manual CAD modeling |
| Create a custom bracket/adapter from specs | LLM-driven programmatic CAD (FreeCAD MCP + text spec) |
| 3D print a copy of an existing object | Polycam/RealityScan + manual scaling + mesh cleanup |
| Generate CAD from a photo of a mechanical part | Multi-view + Point2CAD or manual reconstruction (AI not reliable yet) |
| Design a new part from requirements | Text-to-CAD with LLM (FreeCAD/CadQuery) + manual refinement |
| Scale-accurate manufacturing from photos | Professional photogrammetry + calibrated reference + manual CAD |

### What's viable today (May 2026)

**Reliable now**:
- LLM-generated CAD code for mechanical parts with clear specifications
- Smartphone photogrammetry for visualization and rough measurements
- Professional photogrammetry for precision reverse engineering
- Text-to-CAD for simple parts (brackets, adapters, enclosures)

**Improving rapidly** (6-18 months to viability):
- B-rep generation from multi-view images (mmABC, FutureCAD)
- Agent-based CAD generation with verification (CADSmith, TOOLCAD)
- Image-to-CAD for moderate-complexity mechanical parts

**Not yet viable**:
- Single-photo to accurate parametric CAD for complex parts
- Fully automatic scale calibration without references
- End-to-end image-to-manufacturable-CAD for arbitrary objects

## Competing Views and Uncertainty

- **Is NeRF/3DGS the right foundation for CAD?** Some argue neural representations are fundamentally incompatible with parametric CAD requirements (exact geometry, tolerances). Others believe hybrid approaches will bridge the gap.
- **Will LLMs replace traditional CAD?** Unlikely in the near term. LLMs are good at generating CAD code but poor at geometric reasoning and constraint satisfaction. Agent-based approaches with verification are the likely near-term path.
- **How important is scale accuracy for consumer use cases?** For hobbyist 3D printing, approximate scale is often acceptable (parts are scaled to fit during assembly). For professional manufacturing, it's non-negotiable.

## References
1. [COLMAP](https://colmap.github.io/) — Schönberger & Frahm (2016). Structure-from-Motion and Multi-View Stereo.
2. [NeRF](https://arxiv.org/abs/2003.08934) — Mildenhall et al. (2020). Neural Radiance Fields for View Synthesis.
3. [3D Gaussian Splatting](https://arxiv.org/abs/2308.04079) — Kerbl et al. (2023). Real-time Radiance Field Rendering.
4. [Zero-1-to-3](https://arxiv.org/abs/2303.11328) — Liu et al. (2023). Zero-shot One Image to 3D Object.
5. [TripoSR](https://arxiv.org/abs/2403.02151) — Stability AI (2024). Fast 3D Object Reconstruction from a Single Image.
6. [Point2CAD](https://arxiv.org/abs/2405.00123) — (2024). Reconstructing CAD from Point Clouds.
7. [Text2CAD](https://arxiv.org/abs/2306.17115) — (2023). Generating CAD Designs from Text.
8. [mmABC Dataset](https://arxiv.org/abs/2504.02744) — (2025). Multi-modal dataset for AI-Based CAD.
9. [DeepCAD](https://arxiv.org/abs/2105.09492) — Wu et al. (2021). Deep Generative Models for CAD.
10. [CAD2Sketch](https://arxiv.org/abs/2403.08283) — (2024). CAD model sketch generation.
11. [DAG-BRep](https://arxiv.org/abs/2507.04765) — (2025). Graph-based B-rep generation.
12. [CMT](https://arxiv.org/abs/2504.20830) — (2025). Multimodal conditional B-rep generation.
13. [PLLM](https://arxiv.org/abs/2602.12561) — (2026). Pseudo-Labeling LLMs for CAD Program Synthesis.
14. [Clarify Before You Draw](https://arxiv.org/abs/2602.03045) — (2026). Proactive agents for text-to-CAD.
15. [FutureCAD](https://arxiv.org/abs/2603.11831) — (2026). LLM-driven CAD with B-rep grounding.
16. [CADSmith](https://arxiv.org/abs/2603.26512) — (2026). Multi-agent validated CAD generation.
17. [TOOLCAD](https://arxiv.org/abs/2604.07960) — (2026). RL-trained CAD tool-use agents.
18. [Agent-Aided Design](https://arxiv.org/abs/2604.15184) — (2026). Assembly-aware agentic CAD.
19. [CADAM](https://github.com/Adam-CAD/CADAM) — Open-source text/image-to-CAD web app.
20. [Text-to-.step](https://github.com/Sjs2332/Text-to-.step) — Practical FreeCAD-based text-to-STEP.
21. [FreeCAD MCP](https://github.com/contextform/freecad-mcp) — Claude-to-FreeCAD integration.
22. [MCP-FreeCAD](https://github.com/jango-blockchained/mcp-freecad) — AI-FreeCAD integration via MCP.
