---
layout: default
title: Personalized Photo Selection with VLMs and LoRA Preferences
permalink: /musings/personalized-photo-selection-with-vlms/
---

# Personalized Photo Selection with VLMs and LoRA Preferences
_AI-generated research draft. Verify critical claims with primary sources._
Status: Complete
Completed: 2026-04-18T12:00:29+05:30

## TL;DR
- **Verdict: Worth building. The staged-ranker approach is practical and the personalisation component is the differentiator.**
- The best system shape is a **staged ranker**, not a single aesthetic model: technical quality filter → semantic/content filter → personalised preference model → final curation.
- The first two stages (technical + content) are solved problems. The third stage — personalisation — is where the current art lives.
- For the personalisation layer, the most promising approach is a **fine-tuned VLM** (e.g., Qwen2.5-VL or Llama-3.2-Vision with LoRA on user preference data) that scores photos based on learned taste.
- The personalisation model can be trained on user's past selections (which photos they kept/published/shared) plus explicit pairwise preference feedback.
- For vacation photos specifically, the curation task decomposes into familiar sub-problems: dedup, sharpness check, face quality, composition, and "uniqueness" scoring.

## Overview
This note surveys techniques to automatically select the best photos from a large vacation album (hundreds to thousands of shots) using vision-language models (VLMs), personalised preference models, and staged ranking. The goal is to reduce a vacation dump from 2,000 photos to the 40–80 that are actually worth keeping and sharing, with selections that reflect the user's personal aesthetic taste rather than a generic "good photo" score.

## The Architecture

```
[Raw photo album (2000+ photos)]
        |
        v
[Stage 1: Technical Quality Filter]
  - Blur detection (Laplacian variance, edge-based)
  - Exposure check (over/under-exposed)
  - Resolution / compression artifacts
  → Removes ~30–40% (blurry, badly exposed)
        |
        v
[Stage 2: Content & Semantic Filter]
  - Duplicate/near-duplicate detection (perceptual hash + embedding similarity)
  - Face quality (detect → score based on eyes open, smile, sharpness)
  - Object/scene classification ("this is a photo of food/a sign/a wall")
  - VLM-based composition scoring
  → Removes ~40–50% (duplicates, bad subjects, low interest)
        |
        v
[Stage 3: Personalised Preference Model]
  - LoRA fine-tuned on user's historical selections
  - Scores each photo for personal aesthetic preference
  - Can incorporate context: "vacation in Japan" vs "family dinner"
  → Ranks remaining 400–700 photos by personal appeal
        |
        v
[Stage 4: Final Curation]
  - Diversity enforcement (don't pick 20 shots of the same temple)
  - Temporal spread (cover the full trip timeline)
  - Group shots priority (at least one good photo of each person)
  → Selects top 40–80 photos
```

## Stage 1: Technical Quality (Classical CV)

This stage does not need a VLM or any learning-based model. Classical computer vision is faster, cheaper, and just as accurate.

- **Blur detection**: Compute Laplacian variance of grayscale image. Threshold determines sharpness. This is the single highest-yield filter — ~20–30% of vacation photos are blurry.
- **Exposure**: Histogram analysis. Clip shadows (<5th percentile < 10) and highlights (>95th percentile > 245). Mildly underexposed is acceptable (can fix in post); severely blown highlights are harder to recover.
- **Resolution check**: If photos are from multiple sources (phone + camera + WhatsApp forwards), discard anything below target resolution (e.g., < 2MP).
- **JPEG artifact detection**: Blockiness metric if source includes compressed images.

Implementation note: These checks should be fast. With OpenCV, 2,000 photos can be processed in under 30 seconds on a modern laptop.

## Stage 2: Content & Semantic Filter (VLM-Assisted)

### Duplicate Detection

Vacation photos have many near-duplicates: you took 8 shots of the same sunset hoping one would be good.

- **Perceptual hash (pHash)**: Fast, coarse dedup. Group photos with hash distance < threshold.
- **Embedding similarity**: Use CLIP ViT-L/14 or SigLIP to embed each photo. Cosine similarity > 0.92 → near-duplicate. Keep the one with the best technical quality score from Stage 1.

### Face Quality

- Use MTCNN or RetinaFace for face detection.
- For each detected face, score based on:
  - Sharpness of face region (Laplacian variance, localised)
  - Eye openness (landmark-based: eye aspect ratio)
  - Smile detection (landmark-based: mouth aspect ratio)
  - Face size relative to frame (too small = discard in group shots unless it's the only shot of that person)

### VLM-Based Content Scoring

For the remaining photos, a VLM provides a content-level score. This is where the model judges "is this an interesting photo?"

Prompt template:

> You are rating vacation photos. Score this photo from 0 to 100 based on: (1) Subject interest — is this a meaningful subject (person, landmark, activity) vs mundane (wall, floor, receipt)? (2) Composition quality — rule of thirds, leading lines, framing. (3) Emotional content — does this capture a moment worth remembering? Briefly explain your score.

Model choice: **Qwen2.5-VL-7B** or **Llama-3.2-Vision-11B** are strong enough for this task at local inference speeds. GPT-4V/Claude Vision would be better but too expensive for 400+ photos. A good compromise: use a local VLM for bulk scoring, reserve API models for the top 100 candidates.

Caveat: VLMs have known biases in aesthetic scoring — they tend to prefer well-lit, centred, "postcard" compositions. This is why Stage 3 (personalisation) matters.

## Stage 3: Personalised Preference Model

This is where the system learns *your* taste, not generic aesthetic quality.

### Training Data

The most practical source of training data: your existing photo library.

**Positive examples:**
- Photos you've shared (WhatsApp, Instagram, shared albums)
- Photos you've starred/favourited
- Photos you've edited (lightroom, photos app)
- Photos you've printed or framed

**Negative examples:**
- Photos you deleted
- Photos you took but never looked at again
- Photos with very low view counts in your library

**Pairwise preferences** (better signal):
- From a set of near-duplicates, the one you kept vs the ones you deleted
- "Which of these two group photos do you prefer?" — explicit feedback

### Model Approach

**LoRA fine-tuning on a VLM** is the most practical approach:

1. Take a capable open-weight VLM (Qwen2.5-VL-7B, Llama-3.2-Vision-11B, or InternVL2).
2. Add a regression head or classification head for scoring.
3. Fine-tune with LoRA (rank 16–64) on preference pairs:
   - Input: photo + scoring prompt
   - Target: preference score or pairwise ranking

**Training regimen:**
- Base VLM provides generic aesthetic understanding
- LoRA learns the delta between generic "good photo" and "your kind of good photo"
- This is analogous to how diffusion model LoRAs learn style — the base model knows what a photo is, the LoRA learns what *your* photos look like

### What the Personalisation Model Learns

Examples of personal preference signals:
- You prefer candid shots over posed
- You prefer warm colour temperature (golden hour) over cool/clinical
- You prefer environmental portraits (person + context) over tight face crops
- You dislike HDR-heavy photos
- You have a bias toward photos of specific people (your kids > random street performers)
- You prefer landscape orientation for landscapes, portrait for people

These are not universal aesthetic rules — they're personal taste, and they're learnable from your selection history.

### Implementation Details

- LoRA rank: 16–64 (higher = more capacity, harder to train with small datasets)
- Training data needed: ~500–2000 preference pairs for noticeable personalisation
- Augmentation: horizontal flip, small rotations, colour jitter (increases effective dataset size)
- Validation: hold out 10% of preference pairs, check ranking accuracy
- The model doesn't need to output an absolute "aesthetic score" — a relative ranking of "A is better than B" is sufficient for curation

## Stage 4: Final Curation

After ranking by personalised preference score, apply curation constraints:

1. **Diversity**: Cluster embeddings, enforce max K photos per cluster. Prevents 20 shots of the same sunset.
2. **Temporal coverage**: Bin photos by day/hour of trip. Ensure at least N photos per day.
3. **People coverage**: Detect faces, ensure each person who appears in the album has at least 1–2 good photos.
4. **"Hero shot" boost**: Flag photos that scored exceptionally high (>90th percentile) and protect them from diversity culling.
5. **Target count**: User specifies desired album size (40, 80, 120). The system ranks and truncates.

## Tool Landscape

| Stage | Best Open-Source | Notes |
|-------|-----------------|-------|
| Blur/Exposure | OpenCV | Classical CV, no ML needed |
| Face Detection | MTCNN / RetinaFace | RetinaFace is more accurate but slower |
| Face Scoring | MediaPipe Face Mesh | Landmarks for eye/mouth metrics |
| Duplicate Detection | pHash + CLIP embeddings | pHash for speed, CLIP for accuracy |
| Generic Aesthetic VLM | Qwen2.5-VL-7B / Llama-3.2-Vision-11B | Local inference viable |
| Personalisation | LoRA on above VLM | Training code: QLoRA + PEFT/transformers |
| Embedding/Clustering | CLIP ViT-L + FAISS | For diversity and dedup |

## What Works Today vs Research

| Capability | Status |
|-----------|--------|
| Technical quality filtering | Solved (classical CV) |
| Duplicate detection | Solved (pHash + embeddings) |
| Generic aesthetic scoring | Good (VLMs with prompting) |
| Face quality assessment | Good (landmarks + heuristics) |
| Personalised preference from selection history | Promising (LoRA on VLM) but not production-packaged |
| End-to-end vacation curation system | Components exist; integration is the work |
| Generalised "personal aesthetic" model from small data | Active research; LoRA approaches show strongest results |

## Key Papers

| Paper | Year | Relevance |
|-------|------|-----------|
| NIMA (Talebi & Milanfar) | 2018 | Neural image assessment — predicts aesthetic score distribution. Foundation for learned aesthetic scoring, though CNN-based, not VLM. |
| LAION Aesthetics Predictor | 2022 | CLIP-based aesthetic scoring trained on SAC/LAION ratings. Good baseline for generic aesthetic quality. |
| PhotoWCT² (Li et al.) | 2020 | Style transfer for photos. Relevant to understanding "personal style" in photo editing. |
| LoRA (Hu et al.) | 2021 | Low-rank adaptation. The enabling technique for personalised preference models on VLMs. |
| QLoRA (Dettmers et al.) | 2023 | 4-bit quantized LoRA. Makes fine-tuning VLMs feasible on consumer GPUs. |
| BLIP-2 / InstructBLIP | 2023 | Vision-language models that can answer questions about photos. Architectural precursor to current VLMs. |
| PickScore (Kirstain et al.) | 2023 | Text-to-image preference scoring. Demonstrates that preference prediction can be learned from pairwise data. |
| ImageReward (Xu et al.) | 2023 | Learned reward model for image quality. Shows pairwise preference training works for visual domains. |

## Practical Recommendations

1. **Start with the first two stages only.** Technical quality + dedup + basic face scoring already gets you from 2,000 photos to ~500 decent candidates. Build the personalisation model once you have preference data.
2. **Collect preference data passively.** Every time you delete a photo, keep a photo, share a photo, or choose between near-duplicates — that's a training signal. Log it.
3. **Don't over-automate the final selection.** The system should rank and recommend, not delete. The user should always have final say over the curated album.
4. **Use a reference set for calibration.** Maintain a small set of 20–30 photos that you consider "perfect 10/10" for your taste. Use these to calibrate and validate the personalisation model.
5. **The "interestingness" problem is unsolved.** A technically perfect photo of a boring subject (airport carpet) will score well on technical metrics but shouldn't make the album. Content understanding via VLM helps but isn't perfect.
6. **For the training pipeline:** Start with a base VLM checkpoint, add a trainable regression head, freeze the vision encoder and most of the language model, train only the LoRA adapters. This keeps training feasible on a single GPU.

## Sources
- Talebi & Milanfar (2018) "NIMA: Neural Image Assessment" — IEEE TIP
- LAION Aesthetics Predictor v2 (2022) — github.com/christophschuhmann/improved-aesthetic-predictor
- Hu et al. (2021) "LoRA: Low-Rank Adaptation of Large Language Models" — ICLR 2022
- Dettmers et al. (2023) "QLoRA: Efficient Finetuning of Quantized Language Models" — NeurIPS 2023
- Kirstain et al. (2023) "PickScore: A Better Text-to-Image Preference Metric"
- Xu et al. (2023) "ImageReward: Learning and Evaluating Human Preferences for Text-to-Image Generation" — NeurIPS 2023
- Qwen2.5-VL model documentation — Alibaba Cloud
- Llama 3.2 Vision model documentation — Meta AI
- InternVL2 model documentation — OpenGVLab, Shanghai AI Laboratory

_Gaps: Training data requirements for personalisation models need empirical validation. VLM aesthetic scoring biases need systematic characterisation. The "interestingness" problem requires its own research note._

_Generated by Hermes Agent research workflow. Final review by author. Verified 2026-04-18._
