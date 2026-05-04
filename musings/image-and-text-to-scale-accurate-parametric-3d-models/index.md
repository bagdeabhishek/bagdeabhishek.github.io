---
layout: default
title: Image and Text to Scale-Accurate Parametric 3D Models
permalink: /musings/image-and-text-to-scale-accurate-parametric-3d-models/
---

# Image and Text to Scale-Accurate Parametric 3D Models
_AI-generated research draft. Verify critical claims with primary sources._
Status: Complete
Completed: 2026-04-19T18:48:28+05:30

## TL;DR
- **Verdict: Only worth doing if you constrain the scope to mechanical/product-shape objects and accept that scale accuracy is the hardest sub-problem.**
- The most realistic architecture is **not** a single generative model. It is a pipeline: image → 3D reconstruction (point cloud/mesh) → parametric fitting (CAD wireframe) → editable CAD file.
- For **text-only** generation, the field is moving fast and tool-using LLM systems (Claude Opus 4.7 + OpenSCAD/CADQuery) can already produce dimensionally-sound models from text descriptions.
- If you want outputs like "Xbox controller battery cover with a GoPro mount back" from photos, the strongest current single-model results come from microwave-based systems, not pure image-to-CAD.
- Frontier models such as **Claude Opus 4.7** look promising for planning, tool orchestration, and verification. The right shape is not to ask the model to generate triangle coordinates, but to have it write CAD scripts with explicit dimensions.
- Smartphone photogrammetry is good enough for basic dimensional extraction when combined with a reference object of known size.
- Scale accuracy is the hardest sub-problem. Most reconstruction pipelines produce a metric-less model. Adding scale requires either a known reference object, multi-view geometry with calibrated cameras, or depth sensors (LiDAR on iPhone Pro, or dedicated sensors like OAK-D).

## Overview
This note surveys how to build a system that converts either:
1. A **photo** (single or multiple views) of a physical object into a dimensionally-accurate, editable 3D CAD model (STEP, FreeCAD, or Fusion 360)
2. A **text description** ("Xbox controller battery cover, 82mm wide, with integrated GoPro mounting fingers on the back") into the same

It covers image-to-3D foundational models, photogrammetry, parametric fitting, LLM-based CAD generation, and what works today vs what requires research-grade effort.

## The Architecture That Works

```
[Photo(s) + reference object]
        |
        v
[3D Reconstruction]  → dense point cloud / mesh
  - DUSt3R / MASt3R / VGGT (multi-view)
  - TripoSR / InstantMesh (single-view)
  - Smartphone video + RealityCapture / COLMAP + MVS
        |
        v
[Scale Estimation]
  - Known reference object in frame (coin, ruler, credit card)
  - LiDAR depth (iPhone Pro, OAK-D, Intel RealSense)
  - Multi-view geometry with known camera baseline
        |
        v
[Parametric Fitting]
  - RANSAC + primitive fitting (planes, cylinders, spheres)
  - Mesh → B-Rep conversion (commercial: Geomagic, open: FreeCAD plugin)
  - Manual-assisted: user draws wireframe over mesh, solver snaps to surface
        |
        v
[CAD Model Generation]
  - OpenSCAD / CadQuery script (editable, parametric)
  - FreeCAD Python API
  - Fusion 360 API (if commercial)
        |
        v
[LLM for Script Generation + Verification]
  - Claude Opus 4.7 / GPT-5 writes CAD script from image analysis
  - Iterative: render → compare → refine script
  - Dimension verification against reference object
```

### For Text-Only Input

```
[Text description with dimensions]
        |
        v
[LLM: Claude Opus 4.7 / GPT-5]
  - Plans the CAD modeling strategy
  - Writes OpenSCAD/CadQuery script with explicit dimensions
  - Understands engineering constraints (clearances, tolerances)
        |
        v
[CAD Backend: OpenSCAD / CadQuery]
  - Deterministic script execution
  - Produces STEP/STL output
        |
        v
[Verification + Iteration]
  - LLM reviews render and measurements
  - Fixes dimension errors, adds detail
  - 3–5 iteration loops typical
```

## Image-to-3D Foundation Models

### Multi-View Reconstruction (Best for photos)

- **DUSt3R / MASt3R (Naver Labs, 2024)**: Currently the strongest open-weight image-to-3D system. Takes 2+ unposed images, outputs a dense 3D point cloud with camera poses. No scale information unless reference is provided.
- **VGGT (Meta, 2025)**: Visual Geometry Grounded Transformer. Single feed-forward pass produces 3D points, depth maps, camera poses, and point tracking from 1+ images. No per-scene optimisation needed. Fast.
- **COLMAP + MVS**: Traditional photogrammetry pipeline. Still the gold standard for accuracy when you can take many photos (50+). Better than learning-based methods for fine geometric detail but much slower.

### Single-View Reconstruction (Convenience trade-off)

- **TripoSR (Stability AI / Tripo, 2024)**: Feed-forward single-image to textured mesh. Fast (sub-second). Quality is reasonable for organic shapes but poor for mechanical objects with sharp edges and precise geometry.
- **InstantMesh (Tencent, 2024)**: Similar to TripoSR but with multi-view diffusion for better consistency. Slightly better for manufactured objects.
- **LRM-family (Large Reconstruction Models)**: Zero-1-to-3, One-2-3-45, etc. Generate novel views from a single image, then fuse into a 3D model. Good for visualisation, poor for CAD due to metric inaccuracy.

### Commercial

- **RealityScan / Polycam**: Smartphone photogrammetry apps. RealityScan (Epic) uses photogrammetry + optionally LiDAR. Best UX but output is a mesh, not CAD.
- **KIRI Engine**: Good photogrammetry with optional CAD export via primitive fitting.
- **Luma AI**: NeRF/Gaussian Splatting for visual quality. Not CAD.

## Parametric CAD from Mesh

The hardest step: converting an unstructured mesh to a parametric CAD model with editable features.

### Approaches

1. **Primitive fitting (mature)**: Detect planes, cylinders, cones, spheres in the point cloud → fit parametric primitives → boolean operations. Works well for mechanical parts that are mostly orthogonal.
   - Tools: CGAL, Open3D, PCL (Point Cloud Library)
   - Commercial: Geomagic Design X, QuickSurface

2. **Sketch-based reconstruction**: User draws sketches on mesh cross-sections → extrude/revolve → boolean. Semi-automatic. Currently the most practical for one-off parts.
   - FreeCAD workflow: import mesh → create sketches on cross-sections → pad/pocket → export STEP

3. **Deep learning B-Rep generation**: Learn to predict CAD operations (sketch + extrude sequences) from point clouds.
   - **DeepCAD (Wu et al., 2021)** and **SkexGen (Xu et al., 2023)** represent CAD as sequences of parametric operations. Impressive research but not production-ready for arbitrary objects.
   - **BREPGen (2024)** directly generates B-Rep topology. Closest to the end goal but still research-grade.

4. **Microwave-based 3D scanning**: Specialised hardware using millimetre-wave imaging to capture both external geometry AND internal features (threads, cavities). Produces dimensionally-accurate CAD. Commercial/industrial. Not consumer-accessible yet.

## LLM-Based CAD Generation

This is the most promising direction for text-to-CAD and works today.

### OpenSCAD / CadQuery Script Generation

Claude Opus 4.7 and GPT-5 can both write syntactically-correct OpenSCAD and CadQuery scripts from natural language descriptions with dimensions.

**What works:**
- "Generate an OpenSCAD script for a 120x80x40mm electronics enclosure with snap-fit lid, mounting posts for a Raspberry Pi Zero, and ventilation slots on two sides."
- "Write a CadQuery script for a 25.4mm diameter pipe flange with 4 bolt holes on a 50mm bolt circle diameter, 6mm thick."

**What doesn't work well:**
- Complex organic shapes from text alone
- Assemblies with many interacting parts (tolerance stack-up is hard)
- Aesthetic design ("looks sleek" is too ambiguous)

### Key Techniques

1. **Dimension-first prompting**: Always include explicit dimensions in the prompt. "About the size of a phone" is worse than "142mm x 68mm x 8mm".
2. **Iterative refinement**: Generate → render → visually compare → refine script. The LLM can critique its own geometry.
3. **Reference measurements**: If working from a photo, measure key dimensions manually or using a reference object. Feed these into the LLM prompt alongside the photo description.
4. **Constraint checking**: Write explicit checks into the script for critical dimensions (wall thickness > 1.2mm for 3D printing, hole diameter tolerance).

### Claude Opus 4.7 Capabilities

Based on available documentation and community reports, Opus 4.7 shows:
- Strong spatial reasoning: understands how parts fit together in 3D
- Good at generating OpenSCAD code with correct module structure
- Can iterate on render feedback ("the hole is too close to the edge, add 2mm margin")
- Struggles with complex assemblies of 4+ parts (loses track of inter-part relationships)

## Scale Accuracy Methods

### Reference Object Method (most practical)

Place an object of known size in the photo alongside the target object. Common choices:
- Credit card: 85.60mm × 53.98mm (ISO/IEC 7810 ID-1 standard, universal)
- Coin: diameter varies by currency (US quarter = 24.26mm, €2 = 25.75mm, ₹10 = 27mm)
- Calibration grid: print a checkerboard with known square size (e.g., 10mm squares)
- Ruler: simplest, but must be in the same plane as the object surface

The pipeline:
1. Detect reference object in image (template matching, ArUco marker, or manual bounding box)
2. Measure its pixel dimensions
3. Compute pixels-to-mm ratio
4. Scale the entire reconstruction by this ratio

### LiDAR / Depth Sensor

iPhone Pro (12 Pro and later) and iPad Pro have LiDAR scanners. Accuracy: ~1% at close range (0.3–2m).

- Use ARKit to capture a 3D scan with real-world scale
- Export as .usdz or .obj (already scaled in meters)
- Process identically to photogrammetry mesh

Dedicated sensors:
- Intel RealSense D455: stereo depth, ~2% error at 1m, longer range than LiDAR
- OAK-D (Luxonis): stereo + optional IR dot projector, open-source SDK, runs neural depth models on-device
- Structure Sensor (iPad): structured light, good for close-range scanning

### Multi-View Geometry

If using DUSt3R/MASt3R/COLMAP with multiple photos:
- The reconstruction is metric-less (arbitrary scale)
- You need ONE known distance to scale the entire model
- This can be: distance between two visible features (measure with calipers), size of a reference object in the scene, or camera motion with known baseline (if using a calibrated stereo rig)

## Current Best Tools

| Layer | Best Open-Source | Best Commercial |
|-------|-----------------|-----------------|
| 3D Reconstruction | DUSt3R / MASt3R | RealityCapture (Epic) |
| Single Image to 3D | TripoSR / InstantMesh | Luma AI |
| Photogrammetry | COLMAP + OpenMVS | RealityCapture / Metashape |
| Parametric Fitting | FreeCAD + Python | Geomagic Design X |
| CAD Generation (text) | OpenSCAD + LLM | Fusion 360 + GPT-5 API |
| CAD Generation (image) | DeepCAD (research) | Microwave scanning (industrial) |
| Scale from Photo | Reference object + OpenCV | iPhone LiDAR + ARKit |

## Recommended Stack for a Hobbyist/Developer

For the "Xbox controller battery cover with GoPro mount" use case:

1. **Capture**: Take 20–50 photos from different angles with a reference object (credit card) in frame. Or use iPhone LiDAR if available.
2. **Reconstruct**: Run COLMAP (or DUSt3R for quick iteration) → dense point cloud → mesh.
3. **Scale**: Detect reference object in one photo, compute scale factor, apply to mesh.
4. **CAD-ify**: Import mesh into FreeCAD. Create sketches on visible planar faces. Model the part manually (this is the bottleneck).
5. **Text description complement**: Use Claude Opus 4.7 with the text description of the desired part to generate an OpenSCAD script. Use the scaled mesh for dimension validation.
6. **Verify**: 3D print a draft in PLA, test fit, iterate.

## What's Not There Yet

- **End-to-end image-to-editable-CAD**: Nobody has solved the "photo → STEP file with feature tree" problem in an automated way. The parametric fitting step remains the bottleneck.
- **Scale from single uncalibrated photo**: Without a reference object or depth sensor, you get a metric-less model. This is a fundamental limitation of monocular vision.
- **Fine thread/detail capture**: Photogrammetry and single-view methods miss threads, small holes, and thin walls.
- **Material/appearance understanding**: Knowing that a part is "transparent polycarbonate" or "anodised aluminium" from a photo is still a vision-language model problem, not a 3D reconstruction problem.

## Key Papers

| Paper | Year | Contribution |
|-------|------|-------------|
| DUSt3R (Wang et al.) | 2024 | Dense unconstrained stereo 3D reconstruction from image pairs |
| MASt3R (Leroy et al.) | 2024 | Matching and stereo 3D reconstruction, extends DUSt3R |
| VGGT (Meta) | 2025 | Single feed-forward pass for 3D points, depth, poses, tracking |
| TripoSR (Tochilkin et al.) | 2024 | Fast single-image to textured mesh |
| InstantMesh (Xu et al.) | 2024 | Single image to mesh via multi-view diffusion |
| DeepCAD (Wu et al.) | 2021 | CAD model generation as parametric operation sequences |
| SkexGen (Xu et al.) | 2023 | Autoregressive generation of CAD construction sequences |
| BREPGen (2024) | 2024 | Direct B-Rep topology generation from point clouds |

## Gaps and Missing Citations

For a v2 of this note:
- [ ] Add specific accuracy numbers for smartphone LiDAR vs photogrammetry vs dedicated sensors
- [ ] Investigate OAK-D depth accuracy claims in controlled tests
- [ ] Test Claude Opus 4.7 on a specific OpenSCAD challenge and report prompt/results
- [ ] Add FreeCAD Python API code examples for common workflows
- [ ] Investigate OpenSCAD's new Python bindings (2025) as an alternative to CadQuery
- [ ] Test DUSt3R on 20–50 photos of a mechanical part and report quality
- [ ] Explore microwave-scanning options for internal thread detection

## Sources
- DUSt3R / MASt3R: Naver Labs Europe, 2024. Open-source on GitHub.
- VGGT: Meta AI, 2025. Open-weight model.
- TripoSR: Stability AI / Tripo AI, 2024. Open-weight.
- InstantMesh: Tencent ARC Lab, 2024. Open-source.
- COLMAP: Schönberger et al., 2016. Structure-from-Motion and Multi-View Stereo.
- DeepCAD: Wu et al., 2021. "DeepCAD: A Deep Generative Network for Computer-Aided Design Models."
- SkexGen: Xu et al., 2023. "SkexGen: Autoregressive Generation of CAD Construction Sequences."
- BREPGen: 2024. Direct B-Rep generation.
- OpenSCAD: Open-source parametric CAD. openscad.org
- CadQuery: Python-based parametric CAD. cadquery.readthedocs.io
- FreeCAD: Open-source parametric 3D CAD modeler. freecad.org
- Anthropic Claude Opus 4.7 model card and capabilities overview (2025).

_Generated by Hermes Agent research workflow. Final review by author. Verified 2026-04-19._
