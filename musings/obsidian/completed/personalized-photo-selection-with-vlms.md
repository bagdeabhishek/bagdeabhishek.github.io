# Personalized Photo Selection with VLMs and LoRA Preferences

## TL;DR

Vision-Language Models (VLMs) can curate personal photo collections, but generic aesthetic scores don't capture individual taste. The solution: fine-tune a personal preference model using LoRA on a small set of user-ranked photos, then use that model to score and select the best photos from large albums. The VLM sees the image; the LoRA adapter encodes *your* taste. This gives you a lightweight, privacy-respecting curator that learns from as few as 20-50 labeled examples.

## The Problem

Smartphone cameras produce thousands of photos per trip/event. Manual curation is time-consuming and often gets postponed indefinitely. Existing solutions fail because:

- **Generic aesthetic models** (NIMA, LAION aesthetic predictor) optimize for "average" taste—not yours
- **Cloud-based solutions** (Google Photos, Apple Photos) require uploading private photos
- **Rule-based approaches** (face detection, blur detection) miss the subjective "je ne sais quoi" of good photos
- **Batch selection** algorithms don't consider narrative flow or diversity within an album

## Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│ Photo Album │────▶│ VLM Encoder  │────▶│ Preference Model │
│  (N photos) │     │ (SigLIP/CLIP)│     │  (LoRA adapter) │
└─────────────┘     └──────────────┘     └────────┬────────┘
                                                   │
                                          ┌────────▼────────┐
                                          │  Per-Photo Score │
                                          │   (0.0 – 1.0)   │
                                          └────────┬────────┘
                                                   │
                    ┌──────────────────────────────▼──────────────────────────────┐
                    │                    Selection Strategy                        │
                    │  Top-K │ Diversity-Aware │ Narrative-Aware │ Interactive    │
                    └─────────────────────────────────────────────────────────────┘
```

### Components

1. **Image Encoder**: Frozen VLM vision encoder (SigLIP, CLIP, or DINOv2)
   - Encodes each photo into a fixed-dimensional embedding
   - No fine-tuning needed—these embeddings already capture rich visual semantics

2. **Preference Adapter**: LoRA fine-tuned on user's pairwise comparisons
   - Input: image embedding
   - Output: scalar preference score
   - Trained on triplets: (better_photo, worse_photo, context)

3. **Selection Engine**: Post-processing module
   - Top-K: straightforward highest scores
   - Diversity-aware: penalize similar photos to ensure variety
   - Narrative-aware: sequence selection for storytelling
   - Interactive: present borderline cases for quick user approval

## Training Data

### Minimal Setup (20-50 examples)

For each training example, the user provides:
- **Pair of similar photos** (same scene, different composition)
- **Binary choice**: which one is better?
- **Optional context**: "portrait", "landscape", "action shot", "candid"

### Efficient Labeling

- **Tournament ranking**: Elo-style head-to-head comparisons converge faster than absolute scoring
- **Active learning**: Model proposes the most informative pairs for labeling
- **Transfer from public datasets**: Initialize with AVA (Aesthetic Visual Analysis) dataset preferences, then personalize

## LoRA Implementation

```python
# Conceptual architecture
class PreferenceHead(nn.Module):
    def __init__(self, embed_dim=768, lora_rank=8):
        self.lora_A = nn.Linear(embed_dim, lora_rank, bias=False)
        self.lora_B = nn.Linear(lora_rank, 1, bias=False)
        
    def forward(self, image_embedding):
        # image_embedding: [batch, 768]
        x = self.lora_A(image_embedding)  # [batch, 8]
        x = self.lora_B(x)                # [batch, 1]
        return torch.sigmoid(x).squeeze(-1)  # [batch]
```

**Key design choices:**
- LoRA rank 4-16 (higher for more complex preferences)
- Frozen base VLM (no gradient to vision encoder)
- Binary cross-entropy loss on pairwise comparisons
- Lightweight: adapter adds < 0.1% to model parameters

## Selection Strategies

### Top-K
Simplest approach—take the K photos with highest scores. Fast but can produce near-duplicates.

### Diversity-Aware (MMR)
Maximal Marginal Relevance: iteratively select photos that maximize `λ × score − (1−λ) × max_similarity_to_selected`

```python
def mmr_selection(scores, embeddings, k, lambda_param=0.7):
    selected = []
    candidates = list(range(len(scores)))
    
    for _ in range(k):
        mmr_scores = []
        for i in candidates:
            relevance = scores[i]
            redundancy = max(
                cosine_similarity(embeddings[i], embeddings[j]) 
                for j in selected
            ) if selected else 0
            mmr_scores.append(lambda_param * relevance - (1 - lambda_param) * redundancy)
        
        best = candidates[argmax(mmr_scores)]
        selected.append(best)
        candidates.remove(best)
    
    return selected
```

### Narrative-Aware
Group photos by temporal clusters, select key moments, ensure story arc (establishing shot → detail → climax → resolution).

## Evaluation

| Metric | Description |
|--------|-------------|
| **NDCG@K** | Normalized Discounted Cumulative Gain—how well does ranking match user preferences? |
| **Preference accuracy** | On held-out pairwise comparisons, how often does model agree with user? |
| **Coverage** | Does selection represent all important moments in the album? |
| **User satisfaction** | A/B test: time spent curating + satisfaction score |

## Privacy Advantages

- All computation runs locally
- Photo embeddings can be pre-computed once and stored
- LoRA adapter is tiny (few MB)—easy to backup/sync
- No photos leave the device
- Model is useless without your specific photo embeddings

## Practical Setup

### Hardware Requirements
- **Encoding**: Any device with ~4GB RAM (SigLIP runs on mobile)
- **LoRA training**: Laptop with GPU or Apple Silicon (M1+)
- **Inference**: Can run on-device after training

### Workflow

1. **Bulk embedding**: Run VLM encoder on all photos (one-time, offline)
2. **Initial labeling**: 20-30 pairwise comparisons (~5 minutes)
3. **Train LoRA**: ~30 seconds on laptop GPU
4. **Score album**: Near-instantaneous for thousands of photos
5. **Iterate**: Add more labels as needed; LoRA updates incrementally

## Limitations

- **Cold start**: First-use requires labeling effort
- **Context blindness**: Model doesn't understand personal memories ("this blurry photo of grandma matters")
- **Composition over content**: May prefer well-composed photos of boring subjects
- **Album context**: Scoring individual photos loses album-level narrative properties
- **Temporal drift**: Your taste changes; model needs periodic retraining

## Key References

1. **NIMA (2018)**: Neural Image Assessment—CNN for aesthetic quality prediction
2. **LAION Aesthetics**: CLIP-based aesthetic predictor trained on crowd preferences
3. **LoRA (2021)**: Low-Rank Adaptation of Large Language Models (applied here to vision)
4. **DreamSim (2023)**: Learning new perceptual similarity metrics from synthetic data
5. **PickScore (2023)**: Text-to-image preference scoring via human feedback
6. **ImageReward (2023)**: Learning human preferences for text-to-image generation

## Integration Ideas

- **Photo app plugin**: Drop-in replacement for "best photos this week"
- **Event curation**: Wedding/party albums automatically curated to your taste
- **Photo book generation**: Combined with layout algorithms for print-ready albums
- **Social media**: Pre-select candidates for posting (saves scrolling)
- **Family sync**: Merge preferences from multiple family members for shared albums

## Why LoRA Instead of Full Fine-tuning?

1. **Efficiency**: LoRA adapter is ~1-3 MB vs ~500+ MB for full model
2. **Privacy**: Easy to delete/reset preferences without retraining base model
3. **Multi-user**: Each family member gets their own adapter on shared embeddings
4. **Incremental**: Add new examples without catastrophic forgetting
5. **Shareable**: Export your "taste vector" to friends (compare aesthetic compatibility!)

*Research note compiled 2025-2026. See also: personalization in recommender systems, few-shot preference learning.*