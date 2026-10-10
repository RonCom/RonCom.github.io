# Projects

## TailSignal: pet health data platform

Links simulated vet, daycare, grooming and wellness-plan records (F1 0.983 vs 0.904 for the usual rule) and turns them
into products: breed-aware drug-safety signals on 970,167 real FDA dog reports (0 false alarms per 1,000 vs 43),
risk-adjusted clinic scorecards, antibiotic-resistance trends from 26,396 real FDA isolates, a feline kidney
early-warning model (AUC 0.965 with SDMA, stress-tested and checked with a decision curve), a metered data API and a
business case. Every analysis had a pre-registered test; the failures are reported.

[Executive summary](projects/tailsignal/index.md) · [Research paper](projects/tailsignal/paper.md) · [Customer portal demo](projects/tailsignal/portal.md) · [Business case](projects/tailsignal/business-case.md) · [Blog post](blog/posts/tailsignal.md) · [Code on GitHub](https://github.com/RonCom/tailsignal)

## Screening for trading before announcements

Scores abnormal trading in the 20 days before 176,485 earnings announcements and 1,860 acquisitions (SEC EDGAR, FINRA
off-exchange short volume, daily bars with delisted stocks), and tests it against SEC insider-trading charges read from
litigation releases by a local LLM with a hand-checked accuracy gate. Pre-registered and scored once on 2021–2025: the
main test failed (2.82× lift against a 3× pass line), the placebo and alert-budget tests passed, and the failure is
explained: charged trades are a median 0.07% of the stock's volume over the window, too small for daily data.

[Write-up](blog/posts/insider-screen.md) · [Code on GitHub](https://github.com/RonCom/insider-screen)

## Group underwriting and stop-loss pricing

Prices specific and aggregate stop-loss for synthetic employer groups of 50–2,500 lives from member-level Medicare
claims (CMS DE-SynPUF). Tweedie LightGBM for next-year cost, logistic models for high-cost claimants, a Pareto tail
spliced at $100k and Monte Carlo group totals. Five pre-registered rounds on independent samples: group claims came in at
0.995–1.001 of expected, and the 2010 test year failed on a 36% drop in the synthetic data's claims. Kedro pipelines;
dbt features reconciled row for row across Python, DuckDB and Snowflake.

[Write-up](blog/posts/group-underwriting.md) · [Code on GitHub](https://github.com/RonCom/group-underwriting)

## Medicare payment integrity outlier screening

Flags Medicare providers whose billing is unusual against same-specialty peers, and tests whether those flags come before
OIG exclusions. Public CMS data 2016–2024, 595,000 provider-years; AUC 0.71 against later exclusions. Extended to an
audit plan (integer program under an hour budget: 50% vs 32% of later-excluded providers' dollars on held-out years) and a
contextual-bandit audit policy tested on simulated audit findings.

[Write-up](blog/posts/medicare-fwa.md) · [Code on GitHub](https://github.com/RonCom/medicare-fwa)

## Prescriptive care management

Who to contact, with which program, and how to keep learning, on synthetic health-plan members: a deep learning risk
model (GRU + MLP, AUC 0.713 vs 0.704 gradient boosting), uplift models from a randomized pilot, an integer program over
five outreach programs, a contextual bandit, Evidently drift monitoring with MLflow champion/challenger retraining, and
capacity-constrained steering to real North Carolina physical therapists.

[Write-up](blog/posts/care-outreach.md) · [Code on GitHub](https://github.com/RonCom/care-outreach)

## Answering Medicare billing-policy questions with a local RAG system

Cited answers from four CMS manuals (NCCI chapters 1 and 11, Benefit Policy Manual chapter 15, Claims Processing
Manual chapter 5) using local models through Ollama, with hybrid keyword and embedding search over DuckDB. 84% judged
correct vs 50% without retrieval, 12 of 12 out-of-scope questions declined, and an LLM judge calibrated against 37
human labels (correctness agreement 92%, κ 0.68).

[Write-up](blog/posts/policy-rag.md) · [Code on GitHub](https://github.com/RonCom/policy-rag)

## minijev: a calibrated decision model on a laptop GPU

A local rebuild of TypeSafe AI's Jev, a model that answers yes/no, multiple-choice and rubric questions with
probabilities instead of text. Qwen3 with LoRA, trained with a proper scoring rule on synthetic data from a local
teacher, on an 8 GB GPU. Best so far: 81.8% accuracy with calibration error 0.024 on 1,200 held-out questions (from
72.3% / 0.247 untrained). In progress.

[Write-up](blog/posts/minijev.md)

## Data modeling, end to end

A tutorial from operational (OLTP) models to analytics, proven on DuckDB and carried to Postgres and Snowflake. In progress.
