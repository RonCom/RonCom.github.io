# Projects

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

## Data modeling, end to end

A tutorial from operational (OLTP) models to analytics, proven on DuckDB and carried to Postgres and Snowflake. In progress.
