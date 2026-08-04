
# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Subata Khan
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/subata24/ml-internship
- **Date:** August 2026

> Copy this file to `work/capstone_report.md` and fill it in as you build. Sections 1–8
> mirror the Pass / Needs-Work rubric axes, so nothing here is optional. Sections 0 and 9
> are **paper sections**: your deployed research paper must carry both, and they're here so
> you never rebuild them from memory at ship time.

## 0. Abstract

This project investigates whether historical search-performance and content signals can identify pages that are likely to decline in organic search clicks, helping editors prioritize content refresh efforts. Using the FlyRank ML Internship warehouse, page-level features were constructed from February 2026 data and used to predict whether pages experienced a decline in clicks during March 2026. A transparent rule-based baseline, Logistic Regression, and Random Forest models were evaluated using a client-grouped train-test split to prevent information leakage across websites. After correcting a label-construction artifact by excluding pages with fewer than five February clicks, all three approaches performed close to random chance (best AUC = 0.532), indicating that the available features did not provide meaningful predictive signal. The resulting contribution is an evidence-based assessment showing that these signals should not be used for automated refresh prioritization and identifying directions for richer future data and modeling.

## 1. Problem framing

This project supports the decision of **which content pages should be prioritized for editorial review and refresh**. The unit of analysis is a **content page**, represented by a client-content pair and summarized using February 2026 search performance together with static page metadata. The intended output is a **decline-risk score** that ranks pages according to their likelihood of losing organic search clicks in the following month.

If such a ranking is reliable, content editors can focus limited refresh effort on the highest-risk pages first instead of reviewing every page manually. A false positive wastes editorial time refreshing a page that was unlikely to decline, while a false negative allows a genuinely declining page to lose additional organic traffic before it is reviewed.

Machine learning is appropriate because multiple search and content signals may interact in ways that are difficult to capture with a simple rule. This project evaluates whether combining those signals produces better decision support than a transparent baseline while maintaining an honest validation design that avoids data leakage.

## 2. Data safety

This project uses the FlyRank ML Internship warehouse hosted on Hugging Face. Data comes from two public-safe tables: `fact_content_daily_performance`, which contains daily Google Search Console and GA4 performance metrics, and `dim_content`, which contains static page-level metadata such as word count, search volume, and content creation date.

Several fields were deliberately excluded to prevent information leakage and to keep the model focused on information that would realistically be available before a prediction is made. Label-derived fields such as `trend_direction` and `trend_pct` were not used because they directly describe the outcome being predicted. Editorial or system-generated metadata (for example, optimization eligibility or AI-generation indicators) were also excluded because they may encode human decisions rather than organic search behaviour.

The target variable was created only from the February-to-March change in Google Search Console clicks. Features were computed exclusively from February data and static page metadata so that no future information leaked into model training. Client and content hash identifiers were used only for joining tables and for grouped train-test splitting. They were never included as model features.

Pages with unavailable Search Console data were excluded because decline could not be measured reliably. Pages with fewer than five February clicks were also excluded to remove a floor-effect artifact, where pages with almost no traffic could not meaningfully decline and would artificially inflate model performance.

No client names, domains, URLs, search queries, credentials, or other identifying information appear anywhere in this project. All identifiers are salted hash values used only for grouping and joining, making the analysis safe for public release.

## 3. Baseline

Before training machine learning models, I implemented a simple, transparent baseline that mirrors the Week 4 assignment. Each page received a baseline score based on two manually defined signals:

- **Stale content:** Pages older than 90 days received one point.
- **CTR underperformance:** Pages whose click-through rate was more than 0.02 below the expected CTR for their average search position received one point.

The baseline score ranged from 0 to 2, with higher scores indicating pages that might benefit from review. This rule-based approach provides a fair comparison because it uses only information available before the prediction window and is fully interpretable by editors.

The baseline was evaluated on the same client-grouped held-out test set as the machine learning models. It achieved an **AUC of 0.532**, slightly above random chance (0.50) but still indicating very weak discrimination between declining and non-declining pages. Although simple, this baseline ultimately performed better than both the Logistic Regression and Random Forest models tested in this project.

## 4. Model / analysis

This project frames refresh prioritization as a **binary classification** problem. The target variable indicates whether a content page experienced a decline in Google Search Console clicks from February 2026 to March 2026. A page was labeled as **declined (1)** if its total March clicks were lower than its February clicks; otherwise it was labeled **not declined (0)**.

Two supervised machine learning models were evaluated: **Logistic Regression** as a simple linear baseline and **Random Forest** as a non-linear ensemble model capable of capturing more complex relationships between features. Their performance was compared against the transparent rule-based baseline developed in Week 4.

The feature set consisted entirely of information available before the prediction window:

- Average Google Search position (February)
- Organic sessions (February)
- Engaged sessions (February)
- Word count
- Search volume
- Content age (days)
- CTR gap (observed CTR minus expected CTR based on average position)

Several variables were intentionally excluded. Client and content hash identifiers were used only for joining tables and grouped validation, never as predictive features. Label-derived fields, editorial metadata, optimization flags, and any information from March or later months were excluded to prevent target leakage and ensure the model reflected a realistic prediction setting.

The objective of the analysis was not to predict Google's ranking algorithm, but to evaluate whether these historical search-performance and content signals could provide useful decision support for identifying pages likely to decline in organic clicks.

## 5. Evaluation

Model evaluation was performed using a **GroupShuffleSplit**, where all pages belonging to the same client were kept together in either the training or test set. This prevents information leakage across clients and better reflects how the model would generalize to unseen websites rather than unseen pages from the same website. A 70%/30% train-test split was used with a fixed random seed (`random_state=42`).

The held-out test set contained **10,747 pages** from **12 unique clients**, with a decline base rate of **41.8%**. All models were evaluated on this identical split using Area Under the ROC Curve (AUC) as the primary metric. Precision@20, Precision@50, and Precision@100 were also computed to evaluate ranking quality.

| Method | AUC |
|--------|------:|
| Baseline Rule | **0.532** |
| Logistic Regression | **0.521** |
| Random Forest | **0.505** |

Precision@K results are shown below. The overall decline base rate on the held-out test set was **41.8%**, so ranking metrics should be interpreted relative to this baseline.

| K | Baseline | Logistic Regression | Random Forest |
|---:|---:|---:|---:|
|20|0.15|0.80|0.15|
|50|0.38|0.60|0.26|
|100|0.41|0.50|0.35|

The results show that none of the machine learning models meaningfully outperformed either the baseline rule or random chance. The transparent baseline achieved the highest AUC, while Random Forest performed only marginally above random chance despite being substantially more complex.

An earlier version of this project appeared to produce a Random Forest AUC of approximately **0.84**. Investigation showed that this was caused by a floor-effect artifact: pages with zero or near-zero February clicks could never be labeled as declining, allowing the model to exploit the label construction instead of learning genuine predictive patterns. After filtering out pages with fewer than five February clicks, this artificial performance disappeared, demonstrating the importance of careful label design and leakage checks.

Although Logistic Regression achieved **Precision@20 = 0.80 (compared with a 41.8% base rate)**, inspection showed that **14 of the top 20 predictions belonged to a single client**, indicating client concentration rather than broadly applicable ranking ability. Larger ranking thresholds (Precision@50 and Precision@100) fell much closer to the overall base rate, supporting the conclusion that no reliable predictive signal was found.

## 6. Interpretation

The feature importance analysis suggests that **content age** was the strongest variable used by the Random Forest model, followed by **word count**, **CTR gap**, and **average search position**. Organic sessions, search volume, and engaged sessions contributed comparatively little. However, because the Random Forest achieved an AUC of only **0.505**, these feature importance values should not be interpreted as evidence that these variables meaningfully predict future decline. Instead, they simply describe how the model partitioned the available training data.

The most important finding of this project is a **negative result**. After removing the floor-effect artifact by excluding pages with fewer than five February clicks, no meaningful predictive relationship remained between the available features and future click decline. This indicates that the variables tested in this study do not contain enough information to reliably identify pages that will lose organic traffic one month later.

An unexpected outcome was that the simple rule-based baseline slightly outperformed both machine learning models. While the baseline itself was still only marginally better than random chance (AUC = 0.532), its performance suggests that increasing model complexity does not automatically produce better predictions when the underlying features carry limited predictive signal.

This project also highlights the importance of careful validation. Earlier experiments appeared to show excellent model performance, but detailed investigation revealed that the apparent improvement resulted from a flaw in the label construction rather than genuine predictive ability. Correcting this issue transformed the project from an apparently successful model into an honest evaluation demonstrating that the tested signals were insufficient for the task.

Rather than viewing this as a failed model, the result provides useful evidence for future work. It suggests that predicting content decline will likely require additional information beyond page-level metrics, such as historical trends across multiple time windows, competitor movement, SERP feature changes, or backlink evolution.

## 7. Recommendation

Based on the evidence from this study, the tested models should **not** be deployed as an automated content refresh prioritization system. None of the evaluated approaches—the transparent baseline, Logistic Regression, or Random Forest—demonstrated meaningful ability to distinguish declining pages from stable pages on a held-out client-grouped test set. Deploying these models would therefore risk giving editors unwarranted confidence in recommendations that perform little better than random selection.

For current editorial workflows, human review should remain the primary decision-making process. Editors should continue prioritizing pages using business importance, strategic content goals, and domain expertise rather than relying on the predictive models evaluated in this project.

If an automated first-pass filter is still desired, the transparent baseline rule can be used only as a lightweight screening tool rather than a ranking engine. Its simplicity and interpretability make it suitable for highlighting potentially stale or underperforming pages, but its AUC of 0.532 indicates that it should not be used to determine refresh priority on its own.

The strongest recommendation from this work is to improve the available data rather than increase model complexity. Future iterations should incorporate longer historical time windows, competitor movement, SERP feature changes, backlink trends, and additional behavioural signals that may better explain future changes in organic search performance.

Confidence in these recommendations is **moderate** because they are supported by consistent evaluation across all tested models using a client-grouped validation design. However, the conclusions are limited to the available warehouse features and the February-to-March evaluation window. Different datasets or richer features may produce different outcomes.

## 8. Reproducibility

The complete implementation for this project is available in the accompanying GitHub repository. All notebooks used throughout the internship are committed under the `work/notebooks/` directory, with the capstone workflow implemented in `capstone.ipynb`. The deployed research paper is linked through `submission/paper_url.txt`, as required by the capstone specification.

To reproduce the results from a fresh clone:

1. Clone the repository.
2. Install the required Python dependencies (DuckDB, Hugging Face Hub, pandas, NumPy, scikit-learn, matplotlib, and related packages).
3. Configure a valid Hugging Face access token (`HF_TOKEN`) with permission to access the FlyRank internship warehouse.
4. Open and run `work/notebooks/capstone.ipynb` from top to bottom without modification.

Environment used:

- Python 3.11
- duckdb
- pandas
- numpy
- scikit-learn
- matplotlib
- huggingface_hub

Model training uses a fixed random seed (`random_state=42`) for both the GroupShuffleSplit validation and the machine learning models to ensure reproducibility. All evaluation metrics reported in this report were generated from a fresh execution of the notebook on the held-out client-grouped test set.

The repository also includes the figures generated during evaluation (`work/figures/`) and the complete notebook used to produce every table, metric, and chart reported in this capstone.

## 9. Acknowledgments & data credit

This project was built using the **FlyRank ML Internship dataset**, provided as part of the FlyRank Machine Learning Internship.

Data credit: https://flyrank.ai

The analysis, modeling, evaluation, and conclusions presented in this report are my own and were produced using the public-safe internship warehouse while following the internship's guidelines regarding privacy, responsible reporting, and non-disclosure of client-identifying information.
---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
