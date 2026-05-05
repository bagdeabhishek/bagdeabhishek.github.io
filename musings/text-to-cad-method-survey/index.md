# Text-to-CAD Method Survey: From Smartphone Photos to 3D-Printable Models
_AI-assisted research synthesis. Verify critical claims with primary sources._
Status: Completed
Last updated: 2026-05-05T02:30:00+05:30
Mode: method-survey

## Summary
- Text-to-CAD and image-to-CAD generation has advanced rapidly in 2025-2026, with CadQuery emerging as the dominant intermediate representation.
- Zero-to-CAD (Autodesk AI Lab, Apr 2026) is the most comprehensive reference: 1M synthetic CadQuery programs, Apache 2.0, released model and datasets on Hugging Face.
- For real-world smartphone photo to CAD reconstruction, SAM3 (Meta, Nov 2025) is the key enabler — it isolates objects from backgrounds via text prompts, closing the synthetic-to-real gap.
- A tool-based LLM harness (SAM3 → VLM → CadQuery) is the recommended approach over fine-tuning: no training data needed, components are swappable, each stage is independently debuggable.
- A working prototype is feasible in ~5 days. Start with Zero-to-CAD VL (Apache 2.0, 2.1B params), fall back to CADReasoner (iterative refinement) if needed. Scale via credit card homography.

## Overview

This survey examines methods for generating editable, parametric CAD models from images and text, with a practical focus on reconstructing scale-accurate 3D-printable models from smartphone photos of real objects. The motivating use case: take 20-30 photos of calipers with a credit card for scale, and produce a dimensionally accurate STL file ready for 3D printing — with zero manual steps.

The field has seen explosive growth in 2025-2026, driven by the convergence of three trends: LLMs capable of generating executable Python code (CadQuery), vision-language models that can understand geometry from images, and open-vocabulary segmentation models (SAM3) that can isolate objects from arbitrary backgrounds.

## Background

### Why CadQuery Became the Standard

Early text-to-CAD methods generated task-specific command sequences that required training models from scratch. The shift to CadQuery — a Python-based parametric CAD scripting language — changed everything:

- LLMs already excel at Python code generation
- CadQuery code is executable and immediately verifiable (syntax errors caught at runtime)
- Output is human-readable and editable (named parameters, logical construction order)
- Direct export to industry formats: STEP, STL, 3MF
- The B-Rep (Boundary Representation) kernel provides mathematically precise geometry, not triangle approximations

### The Synthetic Data Revolution

A critical breakthrough: training data can be synthesized. Both Zero-to-CAD (1M programs) and CAD-Recode (1M programs) demonstrated that procedurally generated CadQuery programs — validated by execution — produce training data as effective as real CAD construction histories. This eliminates the data scarcity bottleneck that had limited earlier approaches.

## Core Analysis

### Problem Framing

The problem decomposes into five stages:

1. **Object isolation:** Separate the target object from background in all photos
2. **View selection:** Choose a small set of informative views from many photos
3. **CAD generation:** Convert multi-view images into executable CadQuery code
4. **Validation:** Verify the generated CAD matches the input photos
5. **Scale calibration:** Recover metric dimensions from a known reference object

### Pipeline Mental Model

The recommended architecture is a tool-based LLM harness:

```
Smartphone photos (20-30, object + credit card)
  → SAM3: text-prompted segmentation → clean object images
  → View selector: pick 4 best views by angular coverage
  → VLM: 4 masked views → CadQuery Python script
  → Validator: execute → render → compare → iterate
  → Scale calibrator: credit card homography → metric scale
  → Export: STEP, STL
```

This harness approach has several advantages over fine-tuning a single model:
- Each component solves one well-defined problem
- Components are independently testable and swappable
- No training data required (all components are off-the-shelf)
- The LLM orchestrator provides flexibility and error recovery
- Local failures don't cascade (e.g., if SAM3 misses the object, retry with a different prompt)

### Method Families

#### Text-to-CAD (Natural Language → CAD)

Text-to-CAD methods generate parametric models from natural language descriptions. These are the foundation but don't directly address real-object reconstruction.

**Zero-to-CAD** (Autodesk AI Lab, Apr 2026) is the landmark work. It uses an agentic pipeline where a strong LLM (gpt-oss-120b) generates CadQuery code in a feedback loop with execution validation, documentation lookup, and multi-stage geometric verification. The pipeline produced 999,633 executable sequences with broad operation coverage: booleans, fillets, chamfers, shells, lofts, sweeps, and patterns — far beyond the sketch-and-extrude workflows of earlier datasets. The release includes a curated 100K subset, precomputed DINOv3 embeddings with FAISS index, a fine-tuned Qwen3-VL-2B model for image-to-sequence, and inference code. Apache 2.0 license.

What it does: Synthesizes executable CadQuery programs at scale without real construction-history data.
How it helps: Provides training data and a reference architecture for the entire field.
Why it exists: Real CAD construction histories are proprietary and scarce. Synthetic data breaks this bottleneck.
What it is good at: Scale (1M programs), operation diversity, human-readable output.
What it does not solve well: Real-world photo input. The model was trained on synthetic renders, not smartphone photos.

**CAD-Coder** (NeurIPS 2025) introduced reinforcement learning to CAD code generation. A two-stage pipeline: Supervised Fine-Tuning (SFT) on 8K high-quality text-CadQuery pairs, then Group Reward Policy Optimization (GRPO) with a CAD-specific reward combining Chamfer Distance and format correctness. Uses Qwen2.5-7B-Instruct as the base model. Dataset: 110K text-CadQuery-3D triplets synthesized from Text2CAD via DeepSeek-V3 annotation. Code and model weights released on Hugging Face. Apache 2.0.

What it does: Applies GRPO reinforcement learning to improve CAD code geometric accuracy.
How it helps: The geometric reward (Chamfer Distance) directly optimizes for 3D shape fidelity, not just text similarity.
What it is good at: Higher validity and lower Chamfer Distance than SFT-only approaches.
What it does not solve well: Real-world photos. Trained on synthetic text descriptions and rendered geometries.

**Text-to-CadQuery** (May 2025) was the first method to propose generating CadQuery directly from text, bypassing intermediate command sequences. Fine-tuned six open-source LLMs of varying sizes on 170K text-CadQuery pairs annotated from the Text2CAD dataset. Best model: Qwen2.5-3B with 69.3% top-1 exact match and 48.6% Chamfer Distance reduction. Code and dataset released.

**Additional text-to-CAD methods:** PR-CAD (progressive refinement with DSL), CAD-Tokenizer (VQ-VAE CAD-specific tokenization), FutureCAD (B-Rep primitive grounding), CAD-RL (multimodal CoT-guided RL), STEP-LLM (direct STEP file generation).

#### Image-to-CAD (Photos → CAD)

Image-to-CAD methods take the next step: generating CAD from visual input. These are directly relevant to the smartphone-to-CAD pipeline.

**CADReasoner** (2026) is the strongest candidate for real-photo input. It introduces iterative self-editing: a VLM (Qwen2-VL-2B) generates a CadQuery program, renders the predicted shape, computes geometric discrepancy against the target, and feeds that feedback back to refine the code over multiple iterations. It accepts both multi-view images and point clouds, fusing them as complementary modalities. A novel scan-simulation pipeline injects realistic scanning artifacts (holes, noise, smoothed edges) during training, making the model more robust to imperfect inputs. Code released at github.com/zhemdi/CADReasoner.

What it does: Iteratively refines CadQuery code using geometric discrepancy feedback from rendered output.
How it helps: Each iteration improves geometric accuracy. The closed loop catches errors a single-pass model would miss.
Why it exists: Real scans are noisy. Single-pass generation is brittle. Iterative refinement mirrors how a human engineer would work.
What it is good at: Reducing Chamfer Distance and Invalidity Ratio over iterations, robustness to scan artifacts.
What it does not solve well: Has not been tested on real smartphone photos with SAM3 masks. The scan simulation is synthetic.

**Zero-to-CAD VL** (Autodesk, Apr 2026) is the image-to-sequence component of Zero-to-CAD. Fine-tuned Qwen3-VL-2B (2.1B params) on 8-view renders of the 1M CadQuery dataset. Achieves 82.1% success rate in-distribution and generalizes to human-designed ABC parts. Apache 2.0 license, model weights on Hugging Face (`ADSKAILab/Zero-To-CAD-Qwen3-VL-2B`).

What it does: Converts multi-view rendered images into executable CadQuery programs.
How it helps: Lightest model (2.1B), most open license (Apache 2.0), ready to use.
What it is good at: Fast inference, clean architecture, fine-tunable on custom view counts.
What it does not solve well: Expects 8 views (needs adaptation for 4), no iterative refinement, synthetic-only training.

**cadrille** (ICLR 2026) is a multimodal CAD reconstruction model that handles point clouds, images, and text within a unified VLM framework. Trained on CAD-Recode's 1M synthetic dataset with online reinforcement learning. SOTA on DeepCAD, Fusion360, and CC3D benchmarks. Code released at github.com/col14m/cadrille. A practical pipeline exists: image → InstantMesh → point cloud → cadrille → CadQuery.

What it does: Unified VLM that generates CadQuery from point clouds, images, or text.
How it helps: Multimodal flexibility means it can accept photogrammetry output or direct images.
What it is good at: SOTA benchmark performance, RL training, code available.
What it does not solve well: RL fine-tuning code not released. Real photo domain gap not addressed.

**CADCrafter** (CVPR 2025) is the only method tested on real 3D-printed objects. It generates CAD command sequences from unconstrained images using geometric features (depth and normal maps) to bridge the synthetic-to-real domain gap. A multi-view to single-view distillation enables single-image inference. Trained solely on synthetic DeepCAD data, tested on RealCAD — a dataset of 150 3D-printed CAD models captured with a smartphone. No code or model released.

What it does: Generates CAD from unconstrained real-world photos of 3D-printed objects.
How it helps: Geometric feature encoding (depth/normal) is invariant to surface texture, enabling sim-to-real transfer.
What it is good at: Real photo generalization proven experimentally.
What it does not solve well: No code or model available. Scale is still ambiguous (output varies in size).

**Additional image-to-CAD methods:** Img2CAD (SIGGRAPH Asia 2025, single-view VLM + transformer, code released), GACO-CAD (single image + depth/normal priors + GRPO), GIFT (geometric feedback data augmentation, 1M generated samples), CADEvolve (6-8 canonical views, full CadQuery operator set), DreamCAD (4 ortho views, SOTA but C-UDA license), BrepGaussian (multi-view→Gaussian Splatting→parametric fitting), CADDreamer (single-view→multi-view diffusion→B-Rep).

### SAM3: The Key Enabler

SAM3 (Meta, Nov 2025) is the component that makes the real-photo pipeline viable. It is not a CAD method — it is an open-vocabulary segmentation model. But its ability to isolate objects from arbitrary backgrounds via text prompts solves the single biggest blocker: the synthetic-to-real domain gap.

Key capabilities:
- Text-prompted segmentation: type "caliper" and get pixel-perfect masks in all 20 photos simultaneously
- 848M parameters, 30ms inference per image on H200
- Apache 2.0 license, available on Hugging Face (`facebook/sam3`)
- SAM 3.1 (Mar 2026): 7× faster multi-object tracking
- Can also isolate the credit card scale reference: prompt "credit card"
- Companion SAM3D can generate 3D meshes from single segmented images (experimental)

With SAM3 in the pipeline, real smartphone photos become clean object-only images — much closer to the synthetic renders that all image-to-CAD methods were trained on.

### Scale Calibration

No existing image-to-CAD method produces scale-accurate output. All generate geometrically correct but arbitrarily scaled models. The solution: include a credit card (ISO/IEC 7810 ID-1: 85.60 × 53.98 mm) in the photos.

The calibration process:
1. SAM3 segments the credit card in photos where it is visible
2. From the card's four corners in any frame, compute a 3×3 homography to the canonical metric rectangle
3. Any point in that frame now has a Euclidean distance in millimeters
4. Apply the derived uniform scale factor to the final CadQuery output: `result.val().scale(factor)`
5. Verify by measuring a known feature (e.g., caliper jaw width)

This approach has been validated by tools like VisualRuler, which uses the same credit card homography technique and reports millimeter-level accuracy from a single photo.

### Tradeoffs

The harness approach has clear tradeoffs compared to alternatives:

**Harness vs Fine-tuning:**
- Harness: No training data, swappable components, ~80% automation, moderate accuracy
- Fine-tuning: Needs labeled dataset, higher potential accuracy, brittle to domain shift
- Recommendation: Start with harness, fine-tune specific components when accuracy bottlenecks are identified

**VLM vs Photogrammetry for 3D:**
- VLM: Direct from photos, fast, no point cloud processing, but zero-shot accuracy untested on real photos
- Photogrammetry: Mature, reliable geometry, but slow and produces noisy point clouds that break CAD methods
- Recommendation: VLM first (simpler pipeline), fall back to photogrammetry + manual CAD fitting if needed

**4 views vs 8 views:**
- 4 views: Match DreamCAD and CADCrafter training setups, cover front/back/left/right
- 8 views: Match Zero-to-CAD VL training, may improve accuracy for complex geometry
- Recommendation: Start with 4, add more if validation shows missing features

### Practical Guidance

**For someone building this system today:**

1. Start with SAM3 — it is the foundation. Download `facebook/sam3` from Hugging Face (accept license first). Write a script that takes a folder of photos and a text prompt, outputs masked PNGs.

2. For CAD generation, begin with Zero-to-CAD VL. It is Apache 2.0 licensed, the lightest model (2.1B), and has released inference code. The 8-view training can be adapted: re-render the Zero-to-CAD dataset with 4 orthographic views and fine-tune.

3. Add validation immediately. Execute every generated CadQuery script, render the result from the same viewpoints as the input photos, and compare. A simple Chamfer Distance check catches most failures before they reach the user.

4. The credit card scale module is the simplest piece — OpenCV can detect the card corners, and the homography math is well-understood. Build it last.

5. If Zero-to-CAD VL underperforms on real photos, switch to CADReasoner. Its iterative refinement loop and scan-simulation training make it the most robust option, and the code is available.

**For evaluating whether this approach works for a specific object:**

- Simple prismatic parts (brackets, plates, housings): High confidence. These match the sketch-and-extrude workflows that all methods handle well.
- Parts with fillets, chamfers, and holes: Medium confidence. Covered by Zero-to-CAD's training data, but real photos may obscure fine features.
- Organic or freeform shapes: Low confidence. Current methods are designed for mechanical CAD, not sculptural forms.
- Multi-part assemblies: Very low confidence. No current method handles assemblies.

### Open Questions

1. How well does Zero-to-CAD VL generalize to SAM3-masked real smartphone photos? No one has tested this.
2. Can CADReasoner's iterative refinement compensate for the domain gap, or is fine-tuning on real data necessary?
3. What is the minimum number of views needed for acceptable reconstruction of common mechanical parts?
4. Does the credit card homography method maintain accuracy when the card and object are not in the same focal plane?
5. Can SAM3D (single-image 3D reconstruction) provide a viable shortcut, or is multi-view still necessary for geometric accuracy?

## Evidence and Sources

### Primary: Zero-to-CAD
- Paper: arxiv 2604.24479. Autodesk AI Lab. Agentic synthesis of 1M CadQuery programs.
- Dataset: `ADSKAILab/Zero-To-CAD-1m` (22 likes, 1,260 downloads) and `ADSKAILab/Zero-To-CAD-100k` on Hugging Face.
- Model: `ADSKAILab/Zero-To-CAD-Qwen3-VL-2B` (13 likes, 139 downloads). Image-to-sequence, Qwen3-VL-2B, Apache 2.0.
- Key claim: 82.1% in-distribution success rate for image-to-sequence, generalizes to ABC parts.

### Primary: CADReasoner
- Paper: arxiv 2603.29847. Iterative self-editing for CAD reconstruction.
- Code: github.com/zhemdi/CADReasoner. Qwen2-VL-2B backbone.
- Key claim: Iterative refinement consistently reduces CD and IR over 5 iterations. Scan simulation improves real-world robustness.

### Primary: CAD-Coder
- Paper: arxiv 2505.19713. NeurIPS 2025. SFT + GRPO with geometric reward.
- Model: `gudo7208/CAD-Coder` (2K downloads, 7.6B params). Dataset: `gudo7208/CAD-Coder` (475 downloads).
- Key claim: GRPO with Chamfer Distance reward improves validity and geometric accuracy over SFT-only.

### Primary: cadrille
- Paper: arxiv 2505.22914. ICLR 2026. Multimodal CAD reconstruction with online RL.
- Code: github.com/col14m/cadrille. Qwen2-VL-2B backbone.
- Key claim: SOTA on DeepCAD, Fusion360, CC3D. RL benefits transfer across modalities.

### Primary: CADCrafter
- Paper: arxiv 2504.04753. CVPR 2025. Real photo CAD generation via geometric features.
- Key claim: Only method tested on real 3D-printed objects with smartphone photos. Geometric features enable sim-to-real transfer.

### Primary: SAM3
- Paper: arxiv 2511.16719. Meta, Nov 2025. Open-vocabulary segmentation.
- Model: `facebook/sam3` on Hugging Face. 848M params, Apache 2.0.
- Key claim: 2× accuracy over prior systems on concept segmentation. 30ms/image inference.

### Secondary: Survey and Tooling
- LLMs-CAD-Survey: ACM Computing Surveys, github.com/lichengzhanguom/LLMs-CAD-Survey-Taxonomy.
- earthtojake/text-to-cad: Open-source harness, 1,620 stars, MIT, build123d + React viewer.
- VisualRuler: visualrulerapp.com. Credit card homography for metric measurements from photos.

## Uncertainties and Competing Views

### The Synthetic-to-Real Gap
Every image-to-CAD method is trained on synthetic renders. SAM3 masking narrows this gap but does not eliminate it — real photos have lighting variation, perspective distortion, motion blur, and depth-of-field effects absent from renders. CADReasoner's scan simulation is the most sophisticated attempt to bridge this gap, but it simulates scanning artifacts (holes, noise), not photographic ones. No method has been tested end-to-end on SAM3-masked smartphone photos.

### Benchmark Limitations
All methods evaluate on curated test sets (DeepCAD, Fusion360, ABC). These are mechanical parts with clean geometry. Real-world objects — especially consumer products — have surface textures, logos, wear marks, and non-ideal shapes that are absent from benchmarks. Reported Chamfer Distances and IoU scores may not translate to practical usability.

### Scale Accuracy
The credit card homography approach is proven for 2D in-plane measurements but has not been validated for 3D scale recovery. If the card is not coplanar with the object, perspective effects introduce error. The assumption of uniform scale (one factor for the entire model) may not hold if the VLM reconstruction has anisotropic distortion.

### Harness vs End-to-End
The harness approach trades potential accuracy for flexibility. An end-to-end model trained specifically on smartphone-to-CAD data could outperform the harness on its target domain, but such a model does not exist and would require a custom dataset. The harness can be built today with off-the-shelf components.

## Practical Takeaways

- The building blocks exist. SAM3, Zero-to-CAD VL, and CadQuery can be wired together without training any new models.
- Start with Zero-to-CAD VL. It is the most open, lightest, and easiest to integrate. Fall back to CADReasoner if iterative refinement is needed.
- Include a credit card in every photo set. It costs nothing and solves the scale problem with proven homography math.
- Build validation first, not last. Executing CadQuery and comparing renders to input photos catches errors before the user sees them.
- The harness is ~5 days to prototype and ~80% automated. The remaining 20% (edge cases, complex geometry, poor photos) will require human judgment.

## References

1. [Zero-to-CAD: Agentic Synthesis of Interpretable CAD Programs at Million-Scale Without Real Data](https://arxiv.org/abs/2604.24479) — Primary. Autodesk AI Lab, Apr 2026.
2. [CADReasoner: Iterative Self-Editing for CAD Reconstruction](https://arxiv.org/abs/2603.29847) — Primary. 2026.
3. [CAD-Coder: Text-to-CAD Generation with Chain-of-Thought and Geometric Reward](https://arxiv.org/abs/2505.19713) — Primary. NeurIPS 2025.
4. [cadrille: Multi-modal CAD Reconstruction with Online Reinforcement Learning](https://arxiv.org/abs/2505.22914) — Primary. ICLR 2026.
5. [CADCrafter: Generating CAD Models from Unconstrained Images](https://arxiv.org/abs/2504.04753) — Primary. CVPR 2025.
6. [SAM 3: Segment Anything with Concepts](https://arxiv.org/abs/2511.16719) — Primary. Meta, Nov 2025.
7. [DreamCAD: Scaling Multi-modal CAD Generation using Differentiable Parametric Surfaces](https://arxiv.org/abs/2603.05607) — Primary. Mar 2026.
8. [Text-to-CadQuery: A New Paradigm for CAD Generation](https://arxiv.org/abs/2505.06507) — Primary. May 2025.
9. [Img2CAD: Reverse Engineering 3D CAD Models from Images](https://arxiv.org/abs/2408.01437) — Primary. SIGGRAPH Asia 2025.
10. [GIFT: Bootstrapping Image-to-CAD Program Synthesis via Geometric Feedback](https://arxiv.org/abs/2603.27448) — Primary. 2026.
11. [CAD-Recode: Reverse Engineering CAD Code from Point Clouds](https://cad-recode.github.io/) — Primary. ICCV 2025.
12. [Large Language Models for Computer-Aided Design: A Survey](https://dl.acm.org/doi/10.1145/3787499) — Secondary. ACM Computing Surveys, 2025.
13. [earthtojake/text-to-cad: Open Source Harness](https://github.com/earthtojake/text-to-cad) — Primary (code). MIT, Apr 2026.
14. [VisualRuler: Credit Card Homography Measurement](https://visualrulerapp.com/) — Primary (tool).
15. [Zero-to-CAD Hugging Face Collection](https://huggingface.co/collections/ADSKAILab/zero-to-cad) — Primary (data/model).
16. [CADReasoner GitHub](https://github.com/zhemdi/CADReasoner) — Primary (code).
17. [cadrille GitHub](https://github.com/col14m/cadrille) — Primary (code).
