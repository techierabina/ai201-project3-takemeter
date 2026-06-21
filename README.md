# TakeMeter — Discourse Quality Classifier for r/diabetes

**Author:** Rabina Karki  
**Course:** AI201 — Project 3  
**Date:** June 2026

---

## What I Built

TakeMeter is a fine-tuned text classifier that categorizes posts from r/diabetes into three functionally distinct labels: `personal_experience`, `health_claim`, and `seeking_support`. The classifier was built by fine-tuning DistilBERT on 200 labeled examples and evaluated against a zero-shot Groq LLaMA 3.3-70b baseline.

The core question this project answers: *can a small fine-tuned model learn to distinguish what a diabetes post is doing — narrating, asserting, or asking — well enough to be useful in a real community tool?*

---

## Community Choice

**Community:** r/diabetes (Reddit)

r/diabetes is a public subreddit with over 400,000 members where people living with Type 1, Type 2, and gestational diabetes share experiences, ask questions, and discuss management strategies. It was chosen because posts serve fundamentally different communicative purposes — some are personal narratives, some assert medical facts, and some are calls for community support. These distinctions have real-world utility: a moderation tool, misinformation flagging system, or triage bot for peer support platforms could all benefit from knowing what a post is doing before acting on it.

---

## Label Taxonomy

Three labels were defined, each capturing a distinct communicative function:

---

**`personal_experience`**
The post centers on narrating something that happened to the author — a diagnosis story, a management win or setback, an emotional reaction to living with diabetes. The post is primarily descriptive and retrospective, not asking for advice or making general factual claims.

*Example 1:*
> "Just hit my one-year diaversary. Last year I was in the ER with a blood sugar of 480, absolutely terrified. Today I ran a 5K and my CGM barely blinked. Didn't think I'd ever feel normal again."

*Example 2:*
> "Had a really rough week. Kept going low overnight and I'm exhausted from waking up every 2 hours to check. Nobody around me really gets how tiring this is to manage every single day."

---

**`health_claim`**
The post makes a factual or medical assertion about diabetes — about causes, treatments, diet, medication, symptoms, or outcomes. The claim may be evidence-based or misinformation; what matters is that the post is asserting something as generally true, not just describing personal experience.

*Example 1:*
> "Metformin has been shown in multiple studies to reduce cardiovascular risk in T2D patients beyond its glucose lowering effects. It is criminally underused as a first line treatment."

*Example 2:*
> "Seed oils are the real cause of the insulin resistance epidemic. Switch to beef tallow and your numbers will stabilize within weeks. Big Pharma doesn't want you to know this."

---

**`seeking_support`**
The post is directed at the community — asking for advice, validation, recommendations, or shared experience. The defining feature is that the author is waiting for a response; the post is a question or a request, not a self-contained statement.

*Example 1:*
> "Newly diagnosed T2D here, completely overwhelmed. What do you wish someone had told you in the first month? Any resources or habits that helped you most?"

*Example 2:*
> "My endo wants to switch me from Lantus to Tresiba. Anyone made this switch? Nervous about recalibrating my whole routine."

---

## Data Collection

**Source:** r/diabetes — public posts collected manually by browsing the subreddit and copying post text, supplemented with Claude-assisted synthetic examples reviewed and corrected manually (see AI Usage section).

**Labeling process:** Each post was labeled using the decision rules defined in planning.md. The primary rule: label by the dominant communicative purpose. If the post ends with a question → `seeking_support`. If it asserts something as generally true → `health_claim`. If it narrates the author's own experience → `personal_experience`.

**Label distribution:**

| Label | Count |
|---|---|
| `personal_experience` | 67 |
| `health_claim` | 67 |
| `seeking_support` | 66 |
| **Total** | **200** |

**Three difficult-to-label examples:**

1. *"I was diagnosed 3 years ago and struggled with my A1C — cutting carbs completely fixed it for me. Anyone else try this?"*
   - Could be: `personal_experience`, `health_claim`, or `seeking_support`
   - **Decision:** `seeking_support` — ends in a direct community question; the story is setup, not the purpose

2. *"Berberine is literally just natural metformin. I've been taking it for 6 months and my fasting glucose dropped 30 points."*
   - Could be: `personal_experience` or `health_claim`
   - **Decision:** `health_claim` — the post leads with a general factual assertion; the personal report is evidence for the claim

3. *"I'm so tired of people telling me I gave myself diabetes. I've been T1 since I was 8. It's not a lifestyle choice and it never was."*
   - Could be: `personal_experience` or `health_claim`
   - **Decision:** `personal_experience` — the emotional/personal register dominates; the factual correction is incidental

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace)

**Training setup:**
- Train/validation/test split: 70% / 15% / 15% (140 / 30 / 30 examples)
- Stratified split to preserve label balance across all sets
- Tokenization: max length 256 tokens with truncation

**Hyperparameters:**
- Epochs: 3
- Learning rate: 2e-5
- Batch size: 16
- Weight decay: 0.01
- Warmup steps: 50

**Key hyperparameter decision:** The default learning rate of 2e-5 was kept unchanged. This is the standard starting point for fine-tuning BERT-family models and is well-suited for small datasets where aggressive learning rates risk overfitting. With only 140 training examples, stability was prioritized over speed.

---

## Baseline

**Model:** Groq `llama-3.3-70b-versatile` (zero-shot, no task-specific training)

**Prompt used:**

```
You are classifying posts from r/diabetes, a Reddit community for people living with diabetes.
Assign each post to exactly one of the following categories.

personal_experience: The post narrates something that happened to the author — a diagnosis story, emotional reaction, management win or setback. Primarily descriptive, no question directed at the community.
Example: "Just hit my one-year diaversary. Last year I was in the ER with a blood sugar of 480. Today I ran a 5K and my CGM barely blinked."

health_claim: The post makes a factual or medical assertion about diabetes — about causes, treatments, diet, medication, symptoms, or outcomes. States something as generally true.
Example: "Metformin has been shown in multiple studies to reduce cardiovascular risk in T2D patients beyond its glucose lowering effects."

seeking_support: The post asks the community for advice, validation, recommendations, or shared experience. The defining feature is a question or request directed at others.
Example: "Newly diagnosed T2D here completely overwhelmed. What do you wish someone had told you in the first month?"

Respond with ONLY the label name — nothing else.
Do not explain your reasoning.

Valid labels:
personal_experience
health_claim
seeking_support
```

**How results were collected:** The prompt was run against every example in the locked test set (30 examples) before fine-tuning began. Responses were parsed by matching the model output to a known label string.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq LLaMA 3.3-70b) | **100%** |
| Fine-tuned DistilBERT | **86.7%** |

### Per-Class Metrics — Fine-Tuned Model

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `personal_experience` | 1.00 | 0.50 | 0.67 | 10 |
| `health_claim` | 1.00 | 1.00 | 1.00 | 10 |
| `seeking_support` | 0.67 | 1.00 | 0.80 | 10 |
| **macro avg** | **0.89** | **0.83** | **0.82** | 30 |

### Per-Class Metrics — Baseline

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `personal_experience` | 1.00 | 1.00 | 1.00 | 10 |
| `health_claim` | 1.00 | 1.00 | 1.00 | 10 |
| `seeking_support` | 1.00 | 1.00 | 1.00 | 10 |
| **macro avg** | **1.00** | **1.00** | **1.00** | 30 |

### Confusion Matrix — Fine-Tuned Model

|  | Predicted: personal_experience | Predicted: health_claim | Predicted: seeking_support |
|---|---|---|---|
| **True: personal_experience** | 5 | 0 | 5 |
| **True: health_claim** | 0 | 10 | 0 |
| **True: seeking_support** | 0 | 0 | 10 |

### Analysis of Wrong Predictions

All 5 errors share the same pattern: true label `personal_experience`, predicted `seeking_support`. No other label pair was confused. This is a directional, systematic failure — not random noise.

**Wrong prediction #1:**
> "Been T1D for 22 years now. Some days it still catches me off guard how much mental energy this disease takes. Not looking for advice just needed to say it somewhere people understand."
- True: `personal_experience` | Predicted: `seeking_support` | Confidence: 0.37
- **Analysis:** The phrase "needed to say it somewhere people understand" sounds like it is directed at the community — which is the surface signal the model uses for `seeking_support`. The model missed the explicit disclaimer "not looking for advice" which signals personal venting, not a request.

**Wrong prediction #2:**
> "I was uncontrolled for years. Diet and exercise wasn't helping. My endo put me on GLP1 and I am so satisfied with my results."
- True: `personal_experience` | Predicted: `seeking_support` | Confidence: 0.38
- **Analysis:** This is a short success post with no question. The model may have learned that short posts without strong medical claims default to `seeking_support`. The confidence is very low (0.38), indicating genuine uncertainty.

**Wrong prediction #3:**
> "Finally figured out my dawn phenomenon. Waking up high every day for two years. A small overnight basal increase fixed it in a week. Why did I wait so long to address this."
- True: `personal_experience` | Predicted: `seeking_support` | Confidence: 0.40
- **Analysis:** "Why did I wait so long to address this" is a rhetorical question, not a community question — but the model treated it as a literal question and predicted `seeking_support`. This reveals the model cannot distinguish rhetorical from genuine questions.

**Pattern identified:** The model consistently misclassifies `personal_experience` posts as `seeking_support` when the post contains community-directed language, rhetorical questions, or phrases that sound like requests even when the post is self-contained. All 5 errors had very low confidence (0.36–0.40), meaning the model was uncertain — it just defaulted to the wrong label.

**Why this boundary is hard:** Personal experience posts in patient communities often use community-directed language ("needed to share this with people who understand") without actually asking for anything. The structural signal the model learned for `seeking_support` — community-directed language — overlaps with how personal posts are naturally written on Reddit.

---

### Sample Classifications

| Post (truncated) | True Label | Predicted | Confidence |
|---|---|---|---|
| "Foot care is not optional in diabetes. Peripheral neuropathy makes injuries undetectable..." | `health_claim` | `health_claim` ✅ | 0.99 |
| "Had to leave my nephew's wedding reception early because I went low and couldn't bring it back up..." | `personal_experience` | `personal_experience` ✅ | 0.97 |
| "Newly diagnosed T2D here completely overwhelmed. What do you wish someone had told you..." | `seeking_support` | `seeking_support` ✅ | 0.98 |
| "Been T1D for 22 years now. Some days it still catches me off guard how much mental energy this takes..." | `personal_experience` | `seeking_support` ❌ | 0.37 |
| "Metformin has been shown in multiple studies to reduce cardiovascular risk in T2D patients..." | `health_claim` | `health_claim` ✅ | 0.99 |

The correct prediction of the foot care post is reasonable: the post makes a declarative factual assertion about neuropathy and foot complications without any personal narrative or question, which maps cleanly to `health_claim`.

---

## Reflection: What the Model Learned vs. What I Intended

I intended the model to learn the *communicative function* of each post — what the author was trying to do. What the model actually learned were *surface linguistic signals*: medical vocabulary for `health_claim`, question marks and community-directed phrases for `seeking_support`, and first-person narrative for `personal_experience`.

This works well most of the time because these signals are genuinely correlated with communicative function. But the model fails when the surface signals are misleading — when a personal post uses community-directed language, or when a rhetorical question looks like a genuine one.

The `health_claim` label was easiest to learn (F1 = 1.00) because health claims have the most distinctive vocabulary: clinical terms, hedges like "studies show," and assertive declarative structure. `personal_experience` was hardest (F1 = 0.67) because it shares surface features with `seeking_support` — both are personal, both are written to the community, and the only real distinction is whether the author wants a response.

The model did not learn to detect *intent* — it learned to detect *style*. That gap is fundamental and cannot be fully closed with 200 examples.

---

## Spec Reflection

**One way the spec helped:** The requirement to define hard edge cases before annotating forced me to write explicit decision rules before touching any data. This prevented annotation drift — I applied the same rules consistently throughout because I had written them down. Without this step, the `personal_experience` vs `seeking_support` boundary would have been applied inconsistently and the model would have learned an even noisier signal.

**One way implementation diverged:** The spec assumed data collection would be primarily manual copy-paste from Reddit. Reddit's API restrictions in 2025–2026 made programmatic collection impossible without approved developer access, and manual collection yielded very few `health_claim` examples naturally (most Reddit posts in patient communities are questions or personal stories). I supplemented with Claude-generated synthetic examples for `health_claim`, reviewed and corrected manually. This produced a cleaner, more balanced dataset but one that is less representative of actual Reddit writing style — which likely contributed to the baseline's surprisingly high performance on clean synthetic examples.

---

## AI Usage

**Instance 1 — Dataset generation and labeling:**
I directed Claude to generate 200 realistic r/diabetes-style posts across three labels using my label definitions from planning.md as a guide. Claude produced structured, grammatically clean posts for each label. I then collected real posts from r/diabetes manually and used Claude to label them in batches, providing the full label definitions and decision rules each time. I reviewed every label assignment and corrected several — particularly borderline `personal_experience` vs `seeking_support` cases where Claude consistently applied the decision rule correctly but I occasionally disagreed on tone. Final dataset: 88 real posts, 121 synthetic posts reviewed and accepted.

**Instance 2 — Failure pattern analysis:**
After fine-tuning, I pasted all 5 wrong predictions into Claude and asked it to identify common themes. Claude identified that all errors involved `personal_experience` posts with community-directed language or rhetorical questions, and suggested the model was using surface directedness as a proxy for `seeking_support`. I verified this by re-reading the examples myself — the pattern held. Claude also suggested that low confidence scores (all below 0.40) indicated the model was genuinely uncertain rather than confidently wrong, which I incorporated into my analysis.

**Instance 3 — planning.md structure:**
I asked Claude to help structure planning.md by generating example edge cases for stress-testing my label definitions before annotation. Claude produced 10 boundary posts between `personal_experience` and `seeking_support`. Several of these I could not cleanly classify with my original rules, which prompted me to add the explicit tiebreaker ordering (question → claim → narrative) to the decision rules before annotating.

---

## Repository Structure

```
ai201-project3-takemeter/
├── planning.md                          # Design decisions and annotation log
├── README.md                            # This file
├── diabetes_posts.csv                   # Labeled dataset (200 examples)
├── confusion_matrix.png                 # Confusion matrix from fine-tuned model
├── evaluation_results.json              # Full metrics for both models
└── notebook/
    └── ai201_project3_takemeter_starter_clean.ipynb
```