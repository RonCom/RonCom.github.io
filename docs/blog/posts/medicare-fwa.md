---
date: 2026-09-30
slug: medicare-fwa
authors:
  - chris
categories:
  - Healthcare
  - Anomaly detection
description: Flagging outlier Medicare providers from public CMS data, validated against later OIG exclusions — and why each design choice was made.
---

# Finding outlier Medicare providers with public data

Payment integrity teams have far more claims than reviewers. The practical question is not "is this provider committing fraud?" but "**which providers should a reviewer look at first?**" I built a small, end-to-end pipeline to answer that question using only public CMS data, and tested whether its ranking points toward providers the HHS Office of Inspector General (OIG) later excluded from Medicare.

This post explains the choices behind it: what I built, why, what did not work, and what the results can and cannot support.

<!-- more -->

!!! abstract "TL;DR"
    - **Data:** CMS Medicare Part B and Part D provider summaries, 2016–2024; 594,765 provider-years across three specialties (Interventional Pain Management, Pain Management, Physical Therapists).
    - **Method:** compare each provider to peers in the same specialty and year using robust statistics, adjust for low volume, combine transparent rules with an Isolation Forest, and give every flag plain-language reasons.
    - **Result:** the score ranks providers later excluded by OIG well above chance: **AUC 0.71 (95% CI 0.64–0.76)**. The top 5% of the list holds **3.4×** the random share of later-excluded providers; the top 20% holds 53% of them.
    - **What didn't beat it:** supervised XGBoost, case-mix adjusted peers, and two other outlier detectors — including one that failed a test I wrote down before seeing the data.
    - **Stack:** Python, dbt (running on both DuckDB and Snowflake, reconciled row for row), Streamlit, uv. Code: [github.com/RonCom/medicare-fwa](https://github.com/RonCom/medicare-fwa).

!!! warning "Outliers are not fraud"
    A high score means a provider bills differently from peers. Many will have legitimate reasons (a specialized practice, a sicker population). The score is a way to order a review queue, not a finding. No individual providers are named here.

## Why this problem, and why public data

Fraud, waste and abuse (FWA) in healthcare is mostly found by comparing a provider with similar providers. A physical therapist who bills twice as many 15-minute units per patient visit as nearly every other physical therapist is worth a look, whatever the reason turns out to be. Health plans do this with their own claims. CMS publishes enough aggregated data to do a version of it in the open, which means anyone can check the work.

The data:

| Source | What it gives | Use here |
|---|---|---|
| CMS Physician & Other Practitioners (by provider, and by provider and service) | Services, beneficiaries, payments, and counts by billing code for each provider | Utilization, coding and billing-pattern metrics |
| CMS Part D Prescribers | Drug claims and costs per prescriber | Opioid, long-acting opioid, brand-name and cost metrics |
| NCCI Medically Unlikely Edits (MUE) | CMS's maximum units per code per patient per day | A check on units billed against the limit |
| OIG List of Excluded Individuals/Entities (LEIE) | Providers barred from federal programs, with dates and reasons | The validation label |

I limited the scope to three specialties that often show up in enforcement actions. That kept downloads small (the API is paged with a specialty filter rather than pulling multi-gigabyte files) and let me design metrics that actually mean something for each specialty.

## Design choice 1: compare providers to their own peers

A pain physician and a physical therapist bill completely different codes, and billing norms drift from year to year. So every metric is compared **within specialty × year**. A provider is unusual only if they are unusual relative to people who do the same work in the same year.

Later, this also turned out to matter for validation (below): raw scores are not comparable across specialties, because exclusion rates differ about 20-fold between them.

## Design choice 2: robust statistics, not the textbook bell curve

The classic approach is a z-score: how many standard deviations a provider sits from the mean. Claims metrics are heavily right-skewed, so the extreme billers inflate the mean and standard deviation and partly hide themselves. I kept the classic z and percentile for display, but scored with a **robust z** based on the median and the median absolute deviation (MAD), which the outliers barely move.

![Distribution of services per beneficiary for physical therapists, with the peer median and flagged providers](../../assets/medicare-fwa/peer_distribution_srvcs_per_bene.png)

The chart shows why: the bulk of physical therapists sit in a tight hump, and a long tail stretches far to the right. That tail is where review attention should go, and a mean-based z-score understates how far out it is.

## Design choice 3: don't let small providers dominate

A provider with 15 patients can post an extreme rate by chance. Left alone, the top of the list fills with tiny practices whose numbers are mostly noise. I used **empirical-Bayes shrinkage** (Bühlmann credibility): each provider's rate is pulled toward the peer median in proportion to how little data backs it up.

In plain terms, a rate from 20 patients is trusted less than the same rate from 2,000 patients. The amount of pull (*k*) is estimated from the data for each metric, specialty and year, not set by hand. This is standard in insurance pricing, and it made the top of the list noticeably less dominated by low-volume providers.

## Design choice 4: metrics that map to known schemes

Every metric exists because it corresponds to a recognizable billing problem. That makes a flag explainable to an investigator.

| Metric | Scheme it points to |
|---|---|
| Services per beneficiary; service days per beneficiary | Over-utilization, excess visits |
| Risk-adjusted payment per beneficiary | Cost outliers after case mix and geography |
| Share of level 4–5 office visits | Upcoding |
| Code intensity (units per patient for specific codes) | Excessive units of a procedure |
| Timed units per patient-day | More 15-minute therapy units than a session supports |
| Units per patient-day vs the NCCI MUE | Billing at or above CMS's per-day limit |
| Share of passive modalities; concentration in few codes | Low-value services, narrowed high-yield billing |
| Opioid share, long-acting share, brand share, drug cost | Prescribing risk |

The MUE check is a good example of a small detail that matters. Many codes have an MUE of 1 unit per day, so "at or above the limit" is true for almost everyone and carries no information. I scored only codes whose limit is 2 or more.

## Design choice 5: transparent rules first, a model second

The score has two parts:

1. **Rules:** the average of each provider's three largest robust z-scores. A provider is flagged when it passes 3.5. This is easy to explain: "services per patient and units per visit are both far above peers."
2. **Isolation Forest**, fit separately for each peer group, to catch **unusual combinations** that no single metric shows.

The final score is a **2:1 weighted average** of the two, each first turned into a within-group percentile. I fixed that weighting before looking at validation results, so it could not be tuned to the answer. Every provider carries its top three drivers in plain language, which is what a reviewer actually reads.

![Most common top drivers among flagged providers](../../assets/medicare-fwa/top_drivers.png)

## Design choice 6: a validation label that doesn't cheat

There is no public "confirmed fraud" label. The closest is the OIG exclusion list. I defined a positive as a provider **excluded within three years after the data year**, matched by the same NPI (national provider ID). The ranking is built from year *Y* data and tested against what happened after *Y*, which is how it would be used.

Several traps came up while building this, and catching them changed the numbers more than any model change did:

- **Pooling inflates results.** Scoring all specialties together gave an AUC of 0.78, but mostly because one specialty has both higher scores and far more exclusions. Measured within specialty, the honest number is lower. All results here are within-group.
- **Name matching adds noise.** Some LEIE rows have no NPI. Matching those by name and state added more false matches than true ones (AUC fell to 0.66), so it is reported only as a sensitivity check.
- **The current list forgets people.** The LEIE drops providers once they are reinstated, which would quietly turn some real positives into negatives. I rebuilt a cumulative history from archived snapshots (via the Internet Archive) plus OIG's monthly supplements.
- **Providers repeat across years.** The same provider appears up to nine times. Confidence intervals come from a bootstrap that resamples **providers**, not provider-years, so the same person does not count as independent evidence nine times.

## Results

Over 2016–2024 there are 67 providers excluded within three years of a scored year. That is a small number, so the intervals are wide and I report them everywhere.

| | AUC | Top 5% lift | Top 5% recall |
|---|---|---|---|
| **All three specialties** | **0.71 [0.64–0.76]** | **3.4× [2.1–5.1]** | 17% |
| Fraud-related exclusion types only | 0.72 [0.64–0.79] | 3.6× | 18% |
| Physical Therapists | 0.72 [0.62–0.83] | 5.9× [2.9–9.3] | 29% |
| Pain Management | 0.72 [0.64–0.81] | 2.0× | 10% |
| Interventional Pain Management | 0.66 [0.50–0.78] | 2.6× | 13% |

![AUC by group with 95% confidence intervals](../../assets/medicare-fwa/auc_by_group.png)

Reading it: if you pick one later-excluded provider and one who was not, the score ranks the excluded one higher 71% of the time. A reviewer working the top 5% of the list would reach 17% of later exclusions, 3.4 times what a random 5% would give; working the top 20% reaches 53%.

![Gains curve: share of later-excluded providers captured as the review list grows](../../assets/medicare-fwa/gains_curve.png)

The result holds year by year (AUC between 0.62 and 0.83 for each data year from 2016 to 2024), and it holds for fraud-related exclusion types alone. Utilization metrics carry most of the signal: services per beneficiary, code intensity, units per patient-day and MUE headroom.

One lesson in humility: with only 2021–2024 data, Interventional Pain Management looked like the strongest specialty. Adding 2016–2020 erased that. It was noise from a small sample, which is exactly what the intervals were warning about.

## What didn't work (and why that matters)

I tested ideas from two published papers[^jk][^hamid] against a **frozen** copy of the baseline, rather than folding them into the model as I went. A change would be adopted only if it clearly beat the baseline.

| Idea | AUC | Verdict |
|---|---|---|
| Baseline (rules 2 : Isolation Forest 1) | 0.70 | Kept |
| Supervised XGBoost, cross-validation grouped by provider | 0.65 | Trails baseline |
| XGBoost trained on 2016–2019, tested on 2020–2024 | 0.67 | Trails baseline |
| Peers adjusted for patient case mix | 0.67 | Not adopted |
| Rules + ECOD detector | 0.70 | No gain |
| Rules + CBLOF detector | 0.72 | Failed pre-registered test |

**Supervised learning** improved as labels grew (from near chance with 29 positives to 0.65–0.67 with 67) but still trailed. With 67 positives, a model that learns from labels has little to learn from; an unsupervised peer comparison does not need them. With audit outcomes as labels, that would likely flip.

**CBLOF** looked better than Isolation Forest in three runs on 2021–2024. But those runs shared the same labels, so they were not independent evidence. Before downloading 2016–2020, I wrote down a test in the repository: CBLOF replaces Isolation Forest only if it wins on the new, held-out 2016–2019 years. It did not (AUC difference +0.009, interval −0.013 to +0.032). Isolation Forest stayed. Writing the rule down first is what kept me from adopting a result that did not replicate.

**Why published results look better.** One paper reports much higher accuracy on similar data. I rebuilt its setup on my data and changed one thing at a time:

| Step | AUC |
|---|---|
| Their setup: all specialties pooled, rows split randomly | 0.93 |
| Keep each provider entirely in training or test | 0.82 |
| Rank within specialty-year | 0.66 |
| Predict *future* exclusions | 0.62 |

Most of the headline number came from the same provider appearing in both training and test data, and from pooling specialties with different base rates. Under the same strict test, the unsupervised baseline keeps 0.70. This was the most useful part of the project for me: it shows how much evaluation design, not model choice, drives the number.

I also audited a popular Kaggle "healthcare fraud" dataset as a possible second benchmark. It turned out to be synthetic: one ratio column alone separates the fraud label (AUC 0.99), clinical fields carry no signal, and its provider IDs are not NPIs, so it cannot be linked to real providers. I did not use it.

## Engineering: built like it would be run

- **dbt models run on DuckDB locally and on Snowflake.** The same SQL builds staging tables and a provider-year feature mart in both, with tests for keys and one row per provider-year. A reconciliation step compares every column: all 594,765 provider-years match.
- **Snowflake set up like production:** a service user with key-pair authentication, a dedicated role and small warehouse, and a resource monitor capping spend.
- **Reproducible:** `uv` locks dependencies; one command runs download → load → transform → score → validate. A synthetic-data mode runs the whole pipeline offline in about 30 seconds.
- **A Streamlit dashboard** shows peer distributions, a review queue with drivers, and the method. Providers appear under pseudonymous IDs.

## Limits

- **The label is noisy.** Exclusion lags misconduct by years and captures a small fraction of FWA. Many true problems never lead to exclusion.
- **Aggregates hide claim-level patterns.** Modifier misuse (such as 59/X modifiers to bypass bundling edits), date-of-service patterns and diagnosis coding all need claim lines, which public data does not provide.
- **Small providers are missing.** CMS suppresses counts under 11 beneficiaries.
- **Peers are national.** State or practice-setting peers might change results.
- **67 positives is few.** The intervals are honest, and they are wide.

## What I'd do with a health plan's data

Claim lines would allow modifier and edit-bypass checks, episode-level patterns and referral networks. Audit and recovery outcomes would give a much richer label than exclusions, and at that point a supervised layer on top of these peer features becomes worth building. The core ideas would carry over unchanged: compare to real peers, adjust for volume, keep reasons readable, and evaluate the way the list will actually be used.

The code, results, and full experiment log are at [github.com/RonCom/medicare-fwa](https://github.com/RonCom/medicare-fwa).

[^jk]: Johnson, J. M. & Khoshgoftaar, T. M. (2023). Data-Centric AI for Healthcare Fraud Detection. *SN Computer Science* 4:389.
[^hamid]: Hamid et al. (2024). Healthcare insurance fraud detection using data mining. *BMC Medical Informatics and Decision Making* 24:112.
