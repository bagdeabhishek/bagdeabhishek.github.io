# Image and Text to Scale-Accurate Parametric 3D Models

## TL;DR

Converting images and text descriptions into dimensionally accurate 3D CAD models is hard because most image-to-3D or text-to-3D systems produce visually plausible but metrically wrong meshes. Real engineering needs scale-accurate parametric models (STEP/STL with correct dimensions) that can be fabricated, assembled, and simulated. Recent work combines vision-language models (VLMs) for understanding, foundation models for geometry estimation, and CAD-specific representations (B-rep, CSG, sketch-extrude) to bridge this gap. The key insight: downstream accuracy matters more than photorealism—a simple extrusion from a correct 2D sketch beats a photorealistic mesh that's the wrong size.

## The Problem

Engineering workflows require:
- **Dimensional accuracy**: parts must fit together with specific tolerances
- **Parametric editability**: designs must be modifiable (change thickness, hole size, etc.)
- **Manufacturing-ready output**: STEP for CNC/machining, STL for 3D printing
- **Assembly awareness**: parts must interface correctly

Standard image-to-3D pipelines (NeRF, 3D Gaussian Splatting, DMTet) produce meshes or volumes that:
- Lack consistent scale (how big is this in real units?)
- Have no parametric history (can't edit the design intent)
- Produce noisy topology (not watertight, not manufacturable)

## Key Approaches

### 1. Sketch-Extrude from Images

Systems like **Sketch-to-Parametric-CAD** and **DeepCAD** learn to:
- Detect 2D profile sketches from images
- Infer extrusion parameters (distance, direction, type)
- Generate parametric CAD sequences (Opencascade/FreeCAD commands)

**Pipeline:** Image → Edge Detection → Constraint Solver → Sketch → Extrude → CAD Model

**Strength:** Produces genuine parametric CAD (STEP export, editable features)
**Weakness:** Limited to extrusion-based geometries; struggles with freeform surfaces

### 2. Multi-View Reconstruction + Scale Calibration

Use multiple views (or video) to:
- Recover camera poses via Structure from Motion (SfM)
- Dense reconstruction with MVS (Multi-View Stereo)
- Scale calibration using known reference objects or fiducial markers
- Fit parametric primitives to reconstructed point cloud

**Key paper:** *"Metric3D"* family—learns metric depth from single images by training on mixed datasets with cross-dataset normalization

**Strength:** Handles complex geometries; metric scale possible with reference objects
**Weakness:** Requires multiple views or known references; reconstruction noise propagates to CAD

### 3. VLM-Guided Parametric Fitting

The most promising current direction:
1. **VLM Stage:** Analyze image + text description → understand part type, features, relationships
2. **Measurement Estimation:** Estimate key dimensions from image (using reference scale)
3. **CAD Generation:** Map to parametric templates (e.g., "bearing with ID=X, OD=Y, width=Z")
4. **Iterative Refinement:** Physically-based rendering to compare with input image, gradient-based parameter optimization

**Example:** *"CLAY"* and *"Wonder3D"* generate 3D from images, but the parametric step (CAD fitting) requires additional post-processing like converting meshes to BREP via reverse engineering tools.

### 4. Text-to-CAD via LLMs

Using LLMs to generate CAD modeling code directly:
- **OpenSCAD scripting**: LLM generates `.scad` code from text description
- **CadQuery/FreeCAD Python**: LLM generates parametric Python scripts
- **Zoo/KittyCAD Text-to-CAD API**: Commercial API for text-to-CAD generation

**Evaluation metric:** Chamfer Distance + parametric edit success rate

**Strength:** High editability; natural language interface
**Weakness:** LLMs struggle with spatial reasoning; generated code often has geometry errors

## Critical Dimensions for Real-World Use

| Dimension | Why It Matters | Measurement Method |
|-----------|---------------|-------------------|
| **Scale factor** | Without it, parts are the wrong size | Reference object, known sensor size, depth estimation |
| **Tolerance stack** | Parts must assemble | Explicit tolerance specification in CAD |
| **Material properties** | Affects fabrication parameters | Cannot be inferred from image alone |
| **Surface finish** | Affects fit and function | Cannot be inferred from image alone |
| **Internal features** | Hidden geometry (threads, cavities) | Cannot be seen; must be specified |

## Datasets & Benchmarks

- **ABC Dataset**: Large-scale CAD model dataset with parametric construction sequences
- **Fusion 360 Gallery**: Real-world CAD designs with design timelines
- **DeepCAD**: Paired sketch-extrude sequences for supervised learning
- **Thingi10K**: 10,000 3D-printable models with manufacturing metadata
- **BOP (Benchmark for 6D Object Pose Estimation)**: Objects with accurate 3D models

## Practical Recommendations

1. **Start with reference object**: Always include a ruler, coin, or known-size object in images
2. **Use multi-view**: 3+ images from different angles dramatically improve reconstruction quality
3. **Validate scale early**: Check estimated dimensions against known values before proceeding
4. **CAD template library**: Build a library of common parametric templates (brackets, flanges, etc.)
5. **Human-in-the-loop for critical parts**: Automated systems still need dimensional verification
6. **Combine approaches**: VLM for understanding + traditional photogrammetry for geometry + parametric fitting for CAD

## Key Papers

1. **CLAY (2024)**: 3D-aware diffusion for high-quality mesh generation from images/text
2. **Wonder3D (2023)**: Single image to 3D using cross-domain diffusion
3. **Metric3D v2 (2024)**: Metric depth estimation from single images
4. **DeepCAD (2021)**: A deep generative network for computer-aided design models
5. **SkexGen (2022)**: Autoregressive generation of CAD construction sequences
6. **Text2CAD (2023)**: Generating sequential CAD designs from text prompts
7. **BRepNet (2021)**: Topological deep learning for B-rep classification and segmentation

## Current Limitations

- **Scale remains the hardest problem**: Single-image metric scale estimation is still unreliable
- **Freeform surfaces**: Parametric CAD struggles with organic shapes; hybrid NURBS approaches needed
- **Assembly constraints**: No system handles multi-part assemblies with kinematic constraints
- **Manufacturing validation**: Generated models may not be manufacturable (overhangs, wall thickness, etc.)
- **Performance**: Iterative optimization loops can take minutes per part

## Future Directions

- End-to-end differentiable CAD kernels
- Foundation models for engineering geometry
- Real-time video-to-CAD for AR-assisted design
- Integration with topology optimization for generative design that's both beautiful and structurally sound
- Multi-agent systems where separate models handle understanding, measurement, CAD generation, and validation

## Why This Matters

The gap between "looks right" and "is right" in 3D modeling is vast. Closing it means democratizing manufacturing—enabling anyone with a phone camera to produce dimensionally accurate, editable, manufacturable parts. This is the difference between a 3D-printed toy and a functional replacement part for a machine.

*Research note compiled 2025-2026. Field is moving extremely fast—check recent ArXiv papers for latest developments.*