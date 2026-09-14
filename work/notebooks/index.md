

# FlyRank Content Refresh Intelligence: Predictive Triage Pipeline for High-Impact SEO Optimization

- **Author:** Malik Abdul Rehman
- **Lane:** Content Refresh & Search Intelligence
- **Date:** September 2026

---

## Title & Abstract

The FlyRank search intelligence pipeline addresses organic traffic decay by transforming unstructured search telemetry into a prioritized, decision-support scoring system for content refreshes. By analyzing 30,000 anonymized web pages across historical impression, ranking, and engagement features, we train a supervised Random Forest classifier to detect underperforming search assets with high upside. The model replaces crude heuristic sorting with validated probability scores, establishing an efficient triage system that categorizes pages into distinct operational archetypes. Evaluated across a time-aware validation split, the machine learning pipeline achieves an AUC of 0.84 and a top-decile precision of 0.78, substantially outperforming the heuristic baseline. This automated triage workflow enables editorial teams to prevent click bleed, eliminate review backlogs, and systematically schedule content updates with measurable impact.

---

## 1. Problem framing

Enterprise editorial and growth marketing teams manage tens of thousands of published URLs, but editorial review bandwidth is strictly constrained. Content inevitably suffers from search ranking shifts, algorithmic volatility, and click decay over time. Without automated prioritization, editors are left manually inspecting spreadsheets or arbitrarily selecting URLs for refresh audits.

- **Decision Supported:** The pipeline supports the resource allocation decision of whether an editorial team should assign an editor to refresh meta tags and content for a given URL, place it on a monitoring watchlist, or safely ignore it.
- **Unit of Analysis:** An individual content URL (`content_id`) observed over a trailing 90-day telemetry aggregation window.
- **Output:** A calibrated probability score (`model_score` $\in [0, 1]$), mapped directly into operational action buckets with plain-language diagnostic reason codes.
- **Cost of Errors:**
  - *False Positive (Model flags a healthy page for review):* The operational cost is minor (~2 to 3 minutes of an editor's time to inspect the metrics, confirm the page is healthy, and dismiss the prompt).
  - *False Negative (Model misses a severely decaying page):* The business cost is severe, leading to compounding click losses on high-ranking keywords, search equity erosion, and missed enterprise revenue.
  - *Design Tradeoff:* Because human inspection is relatively cheap while lost organic visibility is expensive, the system is deliberately optimized for high sensitivity and high precision within the top-ranked decile.

---

## 2. Data safety

Data integrity, customer confidentiality, and non-leakage standards were enforced throughout the entire data engineering and modeling lifecycle:

- **Exclusion of Pseudonymous Identifiers:** All categorical IDs (`client_id`, `client_hash_id`, `content_id`) were strictly excluded from model feature sets. These fields were retained exclusively for stratified group splitting, audit tracking, and final queue export.
- **Raw Private Queries & Zero Client Exposure:** Raw search query strings, private query terms, client domains, and confidential business names were completely excluded from the dataset. Exactly 0 client names or identifiable proprietary terms are used or exposed in this repository.
- **Target Leakage Prevention:** Label-derived telemetry fields (including `trend_direction` and `trend_pct`) were strictly prevented from entering the training feature matrix, ensuring that the model does not reverse-engineer its target via direct proxies.
- **Public-Safety Language:** All claims within this paper represent measured, directional observations on historical data for decision support, rather than unsubstantiated causal forecasts of proprietary search engine indexing mechanisms.

---

## 3. Baseline

Prior to deploying machine learning models, editorial teams relied on a static heuristic baseline developed in Week 4:

- **Heuristic Rule:** A deterministic condition flagging URLs ranking on Page 1 (`avg_position <= 10.0`) that exhibited a click-through rate lower than 1% (`ctr < 0.01`), sorted descending by total volume (`impressions_90d`).
- **Fair Comparison:** The baseline rule was benchmarked on the identical evaluation split and metrics as the machine learning model.
- **Performance:** While the heuristic effectively identifies obvious underperformers, it suffers from rigidity:
  - **AUC:** `0.61`
  - **Precision (Top 10%):** `0.45`
  - **Recall:** `0.50`
- **Limitations:** The rule treats position 1 and position 10 symmetrically despite exponential CTR decay across SERP ranks, generates excessive noise in low-impression regimes, and lacks probabilistic confidence ranking.

---

## 4. Model / analysis

To advance beyond static cutoffs, we implemented a supervised machine learning architecture tailored for non-linear metric interactions:

- **Model Architecture:** Random Forest Classifier (`n_estimators=100`, `max_depth=5`, `random_state=42`). Random Forest was selected because ensemble bagging mitigates overfitting on noisy SERP telemetry, accommodates non-linear relationships between search rank and CTR, and requires no parametric feature scaling.
- **Feature Set:**
  1. `avg_position`: Mean organic search ranking position across the trailing 90-day window.
  2. `ctr`: Click-through rate calculated as total organic clicks divided by impressions.
  3. `impressions_90d`: Aggregated 90-day search impression volume serving as an upside scaling weight.
- **Target Definition:** A binary target variable $y \in \{0, 1\}$ denoting confirmed CTR underperformance on high-visibility assets, allowing the classifier to produce a continuous posterior probability $P(y=1 \mid X)$ for queue prioritization.

---

## 5. Evaluation

The model was validated using a time-aware holdout split to emulate production conditions where past search telemetry predicts upcoming optimization cycles. 

### Metrics Comparison Table

| Metric | Heuristic Baseline | Random Forest Model | Relative Improvement |
| :--- | :---: | :---: | :---: |
| **AUC** | 0.61 | **0.84** | **+37.7%** |
| **Precision (Top 10%)** | 0.45 | **0.78** | **+73.3%** |
| **Recall** | 0.50 | **0.72** | **+44.0%** |

The Random Forest model demonstrates substantial discrimination gains across every core dimension. Notably, precision across the top 10% of ranked candidates rose from 0.45 to 0.78, ensuring that editors focusing on the daily top queue encounter actionable underperformance nearly 80% of the time.

![Validation Performance](https://raw.githubusercontent.com/malik-rehman28/my_repo/main/work/notebooks/chart.png)
---

## 6. Interpretation

Model interpretability analysis reveals the underlying signals driving triage recommendations:

- **Feature Importances:**
  - `ctr` (52% importance) and `avg_position` (36% importance) constitute the dominant predictive signals, identifying URLs whose CTR deviates severely below expected SERP position averages.
  - `impressions_90d` (12% importance) serves as an essential magnitude regulator, prioritizing high-traffic pages where minor CTR recovery yields significant click gains.
- **Negative Result on Content Length:**
  - An exploratory hypothesis evaluated whether raw article word count / content length was predictive of CTR recovery.
  - Empirical testing demonstrated zero meaningful correlation ($r = -0.02$) between content length and post-update click recovery.
  - *Key Finding:* "Longer content does not imply higher CTR." Search snippets and title relevance dictate clickability; bloating article word count provides no observed optimization benefit.

---

## 7. Recommendation

We translate validated probability scores into an automated **Action Playbook** for editorial execution:

### The Triage Archetypes

1. **Priority Review (`model_score >= 0.80`):**
   - *Action:* Immediate editorial triage within 48 hours.
   - *Diagnosis:* High visibility paired with critically depressed CTR. Review SERP competitors, optimize meta title tags, and enhance meta description search intent alignment.
2. **Monitor (`0.40 <= model_score < 0.80`):**
   - *Action:* Automated watchlist. Flag for review if metrics deteriorate over a 14-day tracking window.
3. **Auto-Archive (`model_score < 0.40`):**
   - *Action:* Expected performance. Suppress from editorial review queues to eliminate backlog fatigue.

### Operational Guardrails ("The No-Go List")
- **Never Auto-Publish via LLMs:** Machine learning identifies *where* click underperformance exists; it does not write brand-compliant copy. Generative AI must not auto-publish title rewrites without human-in-the-loop review.
- **Auto-Archive $\neq$ Content Deletion:** The "Auto-Archive" designation denotes low priority for CTR fixes; it must never be integrated into automated CMS deletion workflows.

---

## 8. Reproducibility

To re-run the entire pipeline, reproduce model outputs, and generate figures from a clean repository clone:

```bash
# 1. Clone repository
git clone https://github.com/malik-rehman28/my_repo.git
cd my_repo

# 2. Create and activate virtual environment
python -m venv .venv
# On Windows PowerShell:
.\.venv\Scripts\Activate.ps1
# On macOS/Linux:
# source .venv/bin/activate

# 3. Install pinned dependencies
pip install -r requirements.txt

# 4. Execute reference pipeline and notebooks
python scripts/run_all.py
```

- **Random Seeds:** Fixed globally at `random_state=42`.
- **Environment:** Python 3.10+ with `pandas>=2.2`, `numpy>=1.26`, `scikit-learn>=1.4`, and `matplotlib>=3.8`.

---

## Acknowledgments

Built on the FlyRank ML Internship dataset. Visit [https://flyrank.ai](https://flyrank.ai).
