# Chris Lavelle

Data scientist working across statistical modeling, machine learning and the data engineering underneath them:
Python, R, SQL, dbt, Snowflake and AWS. MS in Computer Science (Machine Learning), Georgia Tech. FRM.

This site holds project write-ups and tutorials, written so the choices behind the work are as visible as the results.

## Latest

**[TailSignal: building a pet health data business, end to end](blog/posts/tailsignal.md)**
Linking vet, daycare, grooming and wellness-plan records, then turning them into breed-aware drug-safety signals,
clinic scorecards, a feline kidney early-warning model and a business case, with every test written down before it ran.

**[Answering Medicare billing-policy questions with a local RAG system](blog/posts/policy-rag.md)**
Cited answers from CMS billing manuals with local models, and a measured evaluation: retrieval by question style,
declines on out-of-scope questions, and an LLM judge checked against human labels.

**[Prescriptive care management: who to contact, with what, and how to keep learning](blog/posts/care-outreach.md)**
Deep learning, uplift modeling, integer programming, contextual bandits and drift-triggered retraining on synthetic
health-plan members, checked against a known truth. Where optimization paid off, where it didn't, and why.

**[Finding outlier Medicare providers with public data](blog/posts/medicare-fwa.md)**
Peer-benchmarked outlier detection on 595,000 public CMS provider-years, validated against later OIG exclusions,
then turned into a budgeted audit plan. Why each design choice was made, what failed, and how a published 0.93 AUC fell
to 0.62 under a strict test.

## Projects

| Project | What it is | Links |
|---|---|---|
| TailSignal: pet health data platform | Record linkage (Splink), breed-aware drug safety on FDA data, clinic scorecards, feline kidney early warning, data API, business case | [Write-up](blog/posts/tailsignal.md) · [Code](https://github.com/RonCom/tailsignal) |
| Medicare payment integrity outlier screening | Provider outlier scoring on CMS data; dbt on DuckDB and Snowflake; audit planning (integer program, contextual bandit); Streamlit dashboard | [Write-up](blog/posts/medicare-fwa.md) · [Code](https://github.com/RonCom/medicare-fwa) |
| Prescriptive care management | Deep learning risk, uplift, integer programming, contextual bandits, MLflow + Evidently retraining, provider steering | [Write-up](blog/posts/care-outreach.md) · [Code](https://github.com/RonCom/care-outreach) |
| Policy RAG on CMS billing manuals | Local-LLM RAG (Ollama, DuckDB, hybrid search) with cited answers; evaluation with a calibrated LLM judge | [Write-up](blog/posts/policy-rag.md) · [Code](https://github.com/RonCom/policy-rag) |
| Data modeling, end to end | Tutorial: operational models to analytics, DuckDB → Postgres → Snowflake | In progress |
