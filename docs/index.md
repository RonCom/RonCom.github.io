# Chris Lavelle

Data scientist working across statistical modeling, machine learning and the data engineering underneath them:
Python, R, SQL, dbt, Snowflake and AWS. MS in Computer Science (Machine Learning), Georgia Tech. FRM.

This site holds project write-ups and tutorials, written so the choices behind the work are as visible as the results.

## Latest

**[Which FINRA rules did this conduct break?](blog/posts/finra-enforcement-nlp.md)**
Predicting the rules FINRA charged in 6,198 disciplinary cases from the masked case summary, and a retrieval system over
the FINRA Rulebook. The classifiers tied and their expectations failed; the retrieval system passed all three of its tests.

**[Screening for trading before announcements, tested against SEC charges](blog/posts/insider-screen.md)**
A pre-registered screen for abnormal trading before 176,485 earnings announcements and 1,860 acquisitions, built from
free SEC, FINRA and market data and scored once against later SEC insider-trading charges. Its main test failed, and
the charged trades show why.

**[Pricing stop-loss for employer groups from claims data](blog/posts/group-underwriting.md)**
Next-year cost, high-cost claimants, a spliced Pareto tail and Monte Carlo pricing of specific and aggregate stop-loss
for synthetic employer groups, tested across five independent Medicare populations with every prediction written down
first. Group claims came in at 0.999 of expected on the final holdout.

**[minijev: a calibrated decision model on an 8 GB laptop GPU](blog/posts/minijev.md)**
A model that answers yes/no, multiple-choice and rubric questions with probabilities instead of text, trained with a
proper scoring rule on synthetic data. Calibration error fell from 0.247 to 0.024 on held-out questions.

**[TailSignal: building a pet health data business, end to end](blog/posts/tailsignal.md)**
Linking vet, daycare, grooming and wellness-plan records, then turning them into breed-aware drug-safety signals,
clinic scorecards, a feline kidney early-warning model and a business case, with every test written down before it ran.

[All write-ups](blog/index.md)

## Projects

| Project | What it is | Links |
|---|---|---|
| **Healthcare and insurance** | | |
| Group underwriting and stop-loss pricing | Member cost and high-cost claimant models, spliced Pareto tail, specific and aggregate stop-loss pricing per group; Kedro, dbt on DuckDB and Snowflake, MLflow | [Write-up](blog/posts/group-underwriting.md) · [Code](https://github.com/RonCom/group-underwriting) |
| Medicare payment integrity outlier screening | Provider outlier scoring on CMS data; dbt on DuckDB and Snowflake; audit planning (integer program, contextual bandit); Streamlit dashboard | [Write-up](blog/posts/medicare-fwa.md) · [Code](https://github.com/RonCom/medicare-fwa) |
| Prescriptive care management | Deep learning risk, uplift, integer programming, contextual bandits, MLflow + Evidently retraining, provider steering | [Write-up](blog/posts/care-outreach.md) · [Code](https://github.com/RonCom/care-outreach) |
| Policy RAG on CMS billing manuals | Local-LLM RAG (Ollama, DuckDB, hybrid search) with cited answers; evaluation with a calibrated LLM judge | [Write-up](blog/posts/policy-rag.md) · [Code](https://github.com/RonCom/policy-rag) |
| **Financial markets and regulation** | | |
| Screening for trading before announcements | Abnormal-trading scores from SEC EDGAR, FINRA off-exchange short volume and daily bars; SEC charges read by a local LLM with a hand-checked accuracy gate | [Write-up](blog/posts/insider-screen.md) · [Code](https://github.com/RonCom/insider-screen) |
| FINRA rule classification and Rulebook RAG | TF-IDF, fine-tuned ModernBERT and local-LLM zero-shot on masked case summaries; retrieval over the FINRA Rulebook with an LLM judge | [Write-up](blog/posts/finra-enforcement-nlp.md) · [Code](https://github.com/RonCom/finra-enforcement-nlp) |
| **Animal health** | | |
| TailSignal: pet health data platform | Record linkage (Splink), breed-aware drug safety on FDA data, clinic scorecards, feline kidney early warning, data API, business case | [Full write-up](projects/tailsignal/index.md) · [Paper](projects/tailsignal/paper.md) · [Portal demo](projects/tailsignal/portal.md) · [Code](https://github.com/RonCom/tailsignal) |
| **Models and methods** | | |
| minijev: calibrated decision model | Qwen3 with LoRA trained with a proper scoring rule on an 8 GB GPU; accuracy and calibration on held-out questions | [Write-up](blog/posts/minijev.md) · In progress |
| Data modeling, end to end | Tutorial: operational models to analytics, DuckDB → Postgres → Snowflake | In progress |

[Project details](projects.md)
