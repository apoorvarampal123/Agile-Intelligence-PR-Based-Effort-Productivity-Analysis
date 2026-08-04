# Can We Derive Agile Metrics from Raw GitHub Data?
---

## Overview

Agile teams track velocity and productivity to plan sprints — but these metrics almost always come from manual story-point estimation (planning poker), which is time-consuming and subjective.

This project investigates whether meaningful agile metrics can be **derived automatically from raw GitHub pull-request data**, with zero manual estimation. Using the [AIDev dataset](https://huggingface.co/datasets/hao-li/AIDev) (MSR 2026 Mining Challenge), we build a full data pipeline that computes cycle time, assigns proxy story points, and calculates contributor velocity and productivity — then answers four research questions through four use cases.

---

## Research Questions

| # | Research Question | Use Case |
|---|-------------------|----------|
| RQ1 | Can agile story sizes be derived from PR timestamps, and how does effort vary by task type? | UC1: Effort Trend Analysis |
| RQ2 | Can distinct developer productivity profiles be identified through unsupervised clustering? | UC2: Productivity Clustering |

---

## Dataset

- **Source:** [AIDev — hao-li/AIDev (HuggingFace)](https://huggingface.co/datasets/hao-li/AIDev) · MSR 2026 Mining Challenge · CC-BY 4.0
- **Files used:** `human_pull_request.parquet` + `human_pr_task_type.parquet`
- **Raw PRs:** 6,618 human-authored pull requests across 818 repositories
- **Task types:** `feat`, `fix`, `chore`, `docs`, `refactor`, `build`, `test`, `ci`, `perf`, `style`, `other` (11 categories)

---

## Data Pipeline

The pipeline runs in five sequential steps. Removed records are saved separately at each step for full traceability.

```
Raw dataset (6,618 PRs)
    │
    ├── Step 1: Join tables · compute dev_time_hrs = closed_at − created_at
    │
    ├── Step 2: Drop open PRs (no closed_at) · remove instant closes ≤ 5 min
    │           → Cycle time is undefined for open PRs; imputing a close date
    │             has no principled basis. Instant closes are automated/trivial.
    │
    ├── Step 3: Remove bot accounts · deduplicate on (user, title)
    │
    ├── Step 4: Per-type IQR outlier removal across all 11 task types
    │           → Feature PRs take 3–5× longer than CI/style PRs.
    │             Global IQR would mislabel most CI PRs as "small".
    │             Bounds are computed within each type independently.
    │
    └── Step 5: Quartile binning per type → S / M / L / XL → effort points 1 / 2 / 4 / 8
                → Clean dataset: 3,998 PRs · 1,796 contributors
```

### Story Point Derivation

Traditional agile teams assign story points through planning poker. AIDev has no such labels, so we construct a reproducible proxy:

For **each task type independently**:
1. Sort PRs by `dev_time_hrs`
2. Split at Q1 / Q2 / Q3 (25th, 50th, 75th percentile of that type)
3. Assign effort points: **S = 1 · M = 2 · L = 4 · XL = 8**

**Why per-type?** A 5-hour `feat` PR is medium effort for features (M = 2 pts). A 5-hour `ci` PR is extra-large for CI work (XL = 8 pts). Per-type binning ensures size labels are always relative to the norms of that task category.

### Derived Features

| Feature | Definition |
|---------|-----------|
| `dev_time_hrs` | `closed_at − created_at` (hours) |
| `size_category` | S / M / L / XL via per-type quartile binning |
| `effort_points` | S=1, M=2, L=4, XL=8 |
| `velocity` | Sum of effort points from merged PRs per contributor per month |
| `productivity` | Avg monthly velocity / avg dev time hrs · capped at 20 |

---

## Use Case 1 — Effort Trend Analysis (RQ1)

**Goal:** Analyse how development effort evolves over time at the contributor level.

### Approach

- Computed monthly average `dev_time_hrs` and average effort points for each contributor active in ≥ 2 months
- Fitted a linear trend to each contributor's monthly velocity series → classified as **Rising / Declining / Stable**
- Built an **interactive dual-axis chart** per contributor (Plotly.js + HTML dropdown):
  - Blue line = avg dev time (hrs)
  - Orange dashed = avg effort points
  - Each contributor tagged with their productivity bucket (from UC2)

### Key Finding

Task type is the dominant predictor of cycle time. `feat` and `refactor` PRs have substantially higher median development time than `ci`, `style`, and `docs` PRs — confirming that per-type binning is a substantive requirement, not a convenience. At the contributor level, effort trajectories are heterogeneous: some contributors show rising velocity over time, others declining, others stable.

### Answer to RQ1

> Yes. Agile story sizes can be derived from PR timestamps in a principled, reproducible way. The per-type quartile approach produces balanced size classes (~25% each) and captures genuine variation in task complexity. Effort varies significantly by task type, with `feat` and `refactor` consistently the highest-effort categories.

---

## Use Case 2 — Productivity Clustering (RQ2)

**Goal:** Identify distinct developer productivity archetypes through unsupervised clustering.

### Approach

**Feature selection** — Each of the 1,796 contributors is represented by 5 features:

| Feature | Description |
|---------|-------------|
| `avg_dev_time_hrs` | Average development time per PR |
| `avg_effort_points` | Average story points per PR |
| `avg_monthly_velocity` | Average effort points delivered per month |
| `merge_rate` | Proportion of PRs merged |
| `productivity` | Velocity / dev time, capped at 20 |

Profiles are built row-by-row from the clean dataset (not by averaging already-averaged values) to avoid mean-of-means distortion across contributors with different observation windows. Features are standardised with `StandardScaler`.

**Finding optimal k** — K-Means run over k = 2–40, validated by:
- **Elbow curve** — diminishing returns in inertia beyond k = 7
- **Silhouette score** — peaks at k = 7 (score = 0.50)
- **DBSCAN cross-validation** (ε = 0.8, min_samples = 5) — strong agreement with K-Means

Optimal k = **7**.

**From 7 clusters → 5 productivity buckets** — Each cluster's mean productivity score is computed and mapped to a named bucket:

| K-Means Cluster | Productivity Mean | Bucket |
|:-:|--:|--|
| 6 | 0.0015 | Very Low |
| 3 | 0.0121 | Low |
| 5 | 0.0630 | Low |
| 0 | 0.2384 | Medium |
| 1 | 0.9089 | High |
| 4 | 2.0900 | High |
| 2 | 6.5563 | Very High |

PCA to 2 dimensions is used for visualisation, confirming geometric separation of clusters in the original 5D feature space.

### Key Finding

- **54.4%** of contributors fall in the Low productivity bucket — consistent with the well-documented open-source pattern where a small core drives the majority of output
- High-productivity contributors take **44.8–64.8%** of their PRs in the `feat` category; lower-bucket contributors are chore and fix-heavy
- Productivity is not just about throughput — it also reflects the *type* of work a contributor takes on

### Answer to RQ2

> Yes. Distinct and stable productivity profiles exist in the AIDev contributor population. K-Means (k=7) and DBSCAN both recover a consistent structure: a large low-activity majority and a smaller high-productivity cohort. The five-bucket framing makes these profiles actionable, and the task-type breakdown adds a qualitative dimension beyond raw output counts.

---

## Tech Stack

```
Python 3.x · pandas · scikit-learn · Plotly.js · matplotlib · seaborn
jupyter notebook · HuggingFace datasets
```

---

## Project Structure

```
📦 agile-intelligence-aidev
 ┣ 📓 Final_WORKING_Usecase_1___2.ipynb   ← UC1 and UC2 (this repo)
 ┣ 📊 outputs/
 ┃  ┣ final_effort_point_clean_dataset.parquet
 ┃  ┣ contributor_profiles.parquet
 ┃  ┣ productivity_clusters.parquet
 ┃  ┣ effort_trend_dropdown.html           ← interactive UC1 chart
 ┃  └── productivity_clusters.html         ← interactive UC2 PCA plot
 ┣ 📄 README.md
 └── 📄 requirements.txt
```

---

## My Contributions

- Full data engineering pipeline (joining, cleaning, per-type IQR outlier removal, quartile binning, feature derivation, productivity score with Winsorisation)
- **UC1:** Effort trend analysis · Rising/Declining/Stable contributor classification · interactive Plotly.js dual-axis chart with productivity badge
- **UC2:** Contributor profile construction · K-Means k-selection (elbow + silhouette) · DBSCAN validation · PCA visualisation · cluster-to-bucket mapping

---

## Limitations

- `dev_time_hrs` is elapsed wall-clock time, not active coding time — a PR left open over a weekend accumulates hours without developer activity
- Observation window ends at PR close; work done locally before the first push is not captured
- Story point proxy has not been validated against human-assigned estimates (none exist in this dataset)

---

## References

[1] H. Li et al. AIDev: A Dataset of AI-Assisted Pull Request Development Activity. MSR 2026. https://huggingface.co/datasets/hao-li/AIDev

[2] E. Kalliamvakou et al. The promises and perils of mining GitHub. MSR 2014.

[3] A. Mockus, R. T. Fielding, J. D. Herbsleb. Two case studies of open source software development. ACM TOSEM, 2002.

---

*University of Victoria · CSC 504 · Summer 2026*
