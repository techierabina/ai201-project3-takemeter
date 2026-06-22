# TakeMeter — planning.md
**Project:** Discourse Quality Classifier for r/diabetes  
**Author:** Rabina Karki  
**Date:** June 2026

---

## 1. Community Choice

**Community:** r/diabetes (Reddit)

r/diabetes is a public subreddit with over 400,000 members where people living with Type 1, Type 2, and gestational diabetes share experiences, ask questions, and discuss management strategies. The discourse is ideal for a classification task because:

- Posts serve fundamentally different communicative purposes: some are personal narratives, some assert medical facts (accurate or not), and some are calls for community support
- The community is active enough to yield 200+ examples without scraping rare or specialized threads
- The distinctions between post types have real-world utility : a patient community moderation tool, a misinformation flagging system, or a triage bot for peer support platforms could all benefit from knowing *what a post is doing* before acting on it
- Quality varies meaningfully: a post sharing a lived A1C struggle is structurally different from one claiming that cinnamon cures diabetes, which is different from someone asking the community if their insulin dose sounds right

This makes it a genuinely interesting classification problem is not just stylistic variation, but functional variation in how posts operate within the community.

---

## 2. Label Taxonomy

### Labels (3 total)

---

**`personal_experience`**  
The post centers on narrating something that happened to the author like a diagnosis story, a management win or setback, an emotional reaction to living with diabetes. The post is primarily descriptive and retrospective, not asking for advice or making general factual claims.

*Example 1:*  
> "Just hit my one-year diaversary. Last year I was in the ER with a blood sugar of 480, absolutely terrified. Today I ran a 5K and my CGM barely blinked. Didn't think I'd ever feel normal again."

*Example 2:*  
> "Had a really rough week. Kept going low overnight and I'm exhausted from waking up every 2 hours to check. Nobody around me really gets how tiring this is to manage every single day."

---

**`health_claim`**  
The post makes a factual or medical assertion about diabetes  about causes, treatments, diet, medication, symptoms, or outcomes. The claim may be evidence-based or misinformation; what matters is that the post is asserting something as generally true, not just describing personal experience.

*Example 1:*  
> "Metformin has been shown in multiple studies to reduce cardiovascular risk in T2D patients beyond its glucose-lowering effects. It's criminally underused as a first-line treatment."

*Example 2:*  
> "Seed oils are the real cause of the insulin resistance epidemic. Switch to beef tallow and your numbers will stabilize within weeks. Big Pharma doesn't want you to know this."

---

**`seeking_support`**  
The post is directed at the community asking for advice, validation, recommendations, or shared experience. The defining feature is that the author is waiting for a response; the post is a question or a request, not a self-contained statement.

*Example 1:*  
> "Newly diagnosed T2D here, completely overwhelmed. What do you wish someone had told you in the first month? Any resources or habits that helped you most?"

*Example 2:*  
> "My endo wants to switch me from Lantus to Tresiba. Anyone made this switch? Nervous about recalibrating my whole routine."

---

## 3. Hard Edge Cases & Decision Rules

### The Core Ambiguity: posts that do all three things at once

Many r/diabetes posts blend personal narrative, a factual claim, and a community question. The decision rule is: **label by the primary communicative purpose  what is this post mainly trying to do?**

Apply this tiebreaker in order:

1. **If the post ends with a question directed at the community** → `seeking_support`, regardless of how much personal story precedes it
2. **If the post asserts something as generally true for all diabetics (not just the author)** and there is no question → `health_claim`
3. **If the post is primarily narrating the author's own experience** with no strong claim and no question → `personal_experience`

---

### Documented Edge Cases

**Edge Case 1:**  
> "I was diagnosed 3 years ago and struggled with my A1C — cutting carbs completely fixed it for me. Anyone else try this?"

Could be: `personal_experience` (their story), `health_claim` (low-carb claim), `seeking_support` (asks community).  
**Decision:** `seeking_support` — ends in a direct community question. The story is setup, not the purpose.

---

**Edge Case 2:**  
> "Berberine is literally just natural metformin. I've been taking it for 6 months and my fasting glucose dropped 30 points."

Could be: `personal_experience` (6-month report), `health_claim` (berberine = metformin assertion).  
**Decision:** `health_claim` — the post leads with a general factual assertion ("is literally just natural metformin"). The personal report is evidence offered for the claim, not the point of the post.

---

**Edge Case 3:**  
> "I'm so tired of people telling me I 'gave myself' diabetes. I've been T1 since I was 8. It's not a lifestyle choice and it never was."

Could be: `personal_experience` (emotional vent), `health_claim` (correcting a misconception about T1D causation).  
**Decision:** `personal_experience` — the emotional/personal register dominates; the factual correction is incidental. No question, no general claim framed as advice or information.

---

## 4. Data Collection Plan

**Source:** r/diabetes — public posts and top-level comments from the past 12 months  
**Method:** Manual collection via Reddit's search and "top posts" filter by month; copy-paste into a CSV. No scraping required for 200 examples.

**Target distribution (aim for ~33% per label):**
- `personal_experience`: ~70 examples
- `health_claim`: ~65 examples  
- `seeking_support`: ~65 examples

**Search strategy per label:**
- `personal_experience`: Sort by "top" in r/diabetes, filter for narrative posts with no question mark; look in the Weekly Thread / rant threads
- `health_claim`: Search terms like "actually causes," "studies show," "proven," "myth," "just take," "works for me AND everyone"
- `seeking_support`: Search for posts ending in "?", newly-diagnosed posts, "advice," "anyone else," "what do you use"

**If a label is underrepresented after 150 examples:** collect specifically from threads known to produce that type (e.g., Monday Rant thread for `personal_experience`, misinformation megathreads for `health_claim`) before moving to annotation.

**CSV columns:**
- `text` — the post or comment content
- `label` — one of `personal_experience`, `health_claim`, `seeking_support`

---

## 5. Evaluation Metrics

**Primary metric: per-class F1 score**  
Accuracy alone is misleading if the dataset has even mild class imbalance. F1 (harmonic mean of precision and recall) gives a single number per class that captures both whether the model is over-predicting a label (low precision) and whether it's missing real examples of a label (low recall). I need all three classes to have acceptable F1 — a model that achieves 90% accuracy by always predicting `seeking_support` is useless.

**Secondary metrics:**
- **Confusion matrix** — to identify which specific label pair the model struggles to distinguish (e.g., `personal_experience` vs `seeking_support` is the most likely confusion)
- **Macro-averaged F1** — average F1 across all classes, weighted equally (not by class size), which penalizes poor performance on smaller classes
- **Baseline comparison** — zero-shot Groq LLaMA 3.3-70b vs fine-tuned DistilBERT on the same test set

**Why not just accuracy?**  
If one label ends up at 40% of the dataset despite efforts at balance, a model predicting that label constantly would achieve 40% accuracy while being completely useless. Per-class F1 catches this. Macro F1 gives a single comparable number that doesn't reward majority-class overprediction.

---

## 6. Definition of Success

A classifier is **genuinely useful** for a patient community triage or moderation tool if:

- **Per-class F1 ≥ 0.70 for all three labels** — no label should be a near-total failure
- **Macro F1 ≥ 0.72** — overall balance across classes is solid
- **Fine-tuned model beats zero-shot baseline by at least 10 percentage points** in macro F1 — if it doesn't, fine-tuning added little value and the task may be too easy (or the labels too noisy)

"Good enough for deployment" means a human moderator reviewing the model's classifications would trust them at least 7 out of 10 times for each label type. If the model systematically fails on one class (e.g., always misses `health_claim`), it is not deployable even if overall accuracy looks acceptable.

**What I'll accept as underperformance worth analyzing (not fixing in this project):**  
Posts that mix all three communicative purposes in roughly equal measure — these are genuinely ambiguous and a per-post F1 hit on them is expected. I'll document their failure pattern but won't try to add a fourth label to accommodate them.

---

## 7. AI Tool Plan

### Label stress-testing
Before annotating 200 examples, I pasted my three label definitions and edge case rules into Claude and asked it to generate 10 posts that sit at the boundary between `personal_experience` and `seeking_support` (the most likely confusion pair). Several of these I could not cleanly classify with my original rules, which prompted me to add the explicit tiebreaker ordering (question → claim → narrative) to the decision rules before annotating.

### Annotation assistance
I used Claude to pre-label batches of collected r/diabetes posts by providing my full label definitions and decision rules. Claude assigned one label per post and I reviewed and corrected every assignment before accepting it. I overrode several borderline `personal_experience` vs `seeking_support` cases where I disagreed with how Claude applied the decision rule. Every final label reflects my own judgment.

### Failure analysis
After fine-tuning, I pasted my full list of misclassified test examples into Claude and asked it to identify common patterns. Claude identified that all errors involved the `personal_experience` vs `seeking_support` boundary — posts with substantial personal context surrounding a question confused the model. I verified this pattern by re-reading every wrong prediction myself before writing the evaluation report.

---

## 8. Stretch Features (to be updated)

*Will update this section before beginning any stretch feature.*

- [ ] Inter-annotator reliability (have at least one other person label 30+ examples)
- [ ] Confidence calibration analysis
- [ ] Error pattern analysis (systematic, not just individual failures)
- [ ] Deployed interface (Gradio or simple HTML+JS)

---

## Annotation Log (Hard Cases Encountered)

| Example (truncated) | Labels considered | Decision | Reason |
|---|---|---|---|
| "I was diagnosed 3 years ago...cutting carbs fixed it...Anyone else try this?" | personal_experience, health_claim, seeking_support | seeking_support | Ends with direct community question; story is setup not purpose |
| "Berberine is literally just natural metformin. I've been taking it 6 months..." | personal_experience, health_claim | health_claim | Leads with general factual assertion; personal report is evidence for the claim |
| "I'm so tired of people telling me I gave myself diabetes. I've been T1 since I was 8..." | personal_experience, health_claim | personal_experience | Emotional register dominates; factual correction is incidental |
| "Looking for something small and discreet and not heinously expensive for a medical alert bracelet." | seeking_support, personal_experience | seeking_support | Short declarative request with no question mark; still a community request |
| "Help me with diet pls! I have T2 diabetes and my thyroid has been high for 4 months..." | seeking_support, personal_experience | seeking_support | Explicit help request despite heavy personal context surrounding it |

---

*Last updated: Milestone 3 complete where annotation log filled with hard cases encountered during labeling.*