---
date: 2026-10-02
slug: tailsignal
authors:
  - chris
categories:
  - Animal health
  - Data platforms
  - Evaluation
description: A pet health data platform that links vet, daycare, grooming and wellness-plan records, then turns them into drug-safety signals, clinic scorecards, a feline kidney early-warning model and a business case.
---

# TailSignal: building a pet health data business, end to end

A dog sees a vet for illness, goes to daycare three days a week, gets groomed every six weeks and renews a wellness plan
once a year. Each of those businesses holds a slice of that dog's health, and none of them shares an ID. I built
TailSignal to show how I would turn those slices into one record and then into products people pay for: drug-safety
signals, clinic scorecards, an early-warning model for kidney disease in cats, and a business case for a company that
owns its clinics outright.

<!-- more -->

!!! tip "Full write-up"
    The [full TailSignal write-up](../../projects/tailsignal/index.md) has pages for
    executives, sales, engineering and research, a [research paper](../../projects/tailsignal/paper.md), the
    [business case with an interactive explorer](../../projects/tailsignal/business-case.md) and a
    [customer portal demo](../../projects/tailsignal/portal.md).

!!! abstract "TL;DR"
    - **Data:** simulated partner records (6 systems, 3 file formats) plus real public data: 1.36M FDA adverse-event reports, 26,396 FDA NARMS bacterial isolates and 4,999 UK clinic notes (SAVSNET PetEVAL).
    - **Linking:** matching households first, then pets within each household, reached F1 0.983 against 0.904 for the usual phone-plus-pet-name rule.
    - **Drug safety:** a breed-aware Bayesian model on 970,167 real dog reports raised no false alarms where the standard method raised 43 per 1,000, and ranked the known herding-breed ivermectin risk in the top 30 rather than 235th or lower.
    - **Clinic scorecards:** risk-adjusted complication rates rank clinics reliably at current volume (rank correlation 0.97 with the truth); death rates need about 5,000 procedures per clinic.
    - **Kidney early warning (cats):** two lab visits plus SDMA reach AUC 0.965, hold at 0.96 when hyperthyroidism, dehydration and muscle loss are added, and beat rechecking every senior cat.
    - **Discipline:** every analysis had a written test before it ran. Several failed, and the write-ups say so.
    - **Code:** [github.com/RonCom/tailsignal](https://github.com/RonCom/tailsignal).

## Why synthetic data, and what is real

No one hands clinic records to a portfolio project, so the partner data (clinics, daycare, grooming, wellness plans) is
simulated from a fixed seed. Simulation has one advantage: every method can be checked against a known answer. I
planted a drug side effect, clinic quality differences and kidney decline, then asked whether each method recovered
them. Its weakness is that I wrote the world, so absolute accuracy is optimistic. I treat the comparisons between
methods as the result.

The simulator was calibrated to published studies (Banfield's anesthesia and oral health reports, VetCompass, IRIS
kidney staging, CEPSAF anesthetic mortality), and 26 of 35 validation checks pass, with every miss listed.

Three parts use real data only:

| Source | Records | Used for |
|---|---|---|
| FDA CVM adverse-event reports (openFDA) | 1.36M reports, 970,167 dogs | Drug-safety signals |
| FDA NARMS animal pathogen isolates | 26,396 dog isolates, 2017–2024 | Antibiotic-resistance trends |
| SAVSNET PetEVAL | 4,999 UK first-opinion clinic notes | Testing a text-mining dictionary on real notes |

## Design choice 1: match households before pets

The first job is to recognize the same pet across businesses. A single probabilistic matching model (Splink) over pet
records reached only 0.48 precision: pets in the same household share every owner field, so it merged siblings.
Matching households first, then pets within each household, fixed it.

| Method | Result |
|---|---|
| Same phone and same pet name (the usual rule) | F1 0.904 |
| One-step probabilistic model | Precision 0.48 (siblings merged) |
| **Households, then pets** | **F1 0.983** |

Privacy is enforced in the pipeline: every released record shares its profile with at least 10 others,
and a dbt test fails the build if any product breaks that. 92% of pets can be released; rare breed groups need more
partners. Owner identifiers never reach a data product.

Later, on clinic records without owner details, matching on pet details alone (name, breed, birth
date) fell to 4% precision at ten times the network's size. Owner identifiers belong in data-sharing agreements.

## Design choice 2: pool across breeds for drug safety

The standard way to scan adverse-event reports is disproportionality: does a drug–reaction pair appear more often than
chance? On 970,167 dog reports it raises many false alarms, and it ranks breed-specific risks badly, because each breed
is a small sample.

I used a hierarchical Bayesian model that pools information across breeds, so a breed-specific excess must be
convincing before it is flagged. On shuffled data, where no real signal exists, it raised 0 false alarms per 1,000
pairs against 43 for the standard method. It ranked the known herding-breed (MDR1) sensitivity to ivermectin-type drugs
in the top 30; unpooled methods placed it 235th or lower. Both results replicated on 2020-and-later reports held out in
advance.

It also missed something. Isoxazoline flea drugs and seizures, the subject of an FDA warning in 2018: the excess is real
(about 1.5 times expected) but below the alert threshold I set in advance. The threshold that keeps false alarms near
zero also delays true signals. That trade-off should be the buyer's choice.

## Design choice 3: check text mining on real notes

Confirming a drug signal needs clinic records, and clinic outcomes live in free-text notes. Following a published
veterinary approach, I built a seizure dictionary from the FDA's own reaction terms, expanded it with spelling variants,
and added rules for negation ("no seizures since") and history.

On simulated notes it reached 75% precision. On 4,999 real UK clinic notes it fell to 30%. Nearly all the false hits
came from one word: "fit". The official term "Fit" also means "fit for vaccination", "fit to travel" and "muzzle fits
well" in British clinic English.

I then checked my own labels. PetEVAL's annotators assign each note a diagnosis chapter. Every note I had called a true
seizure carries their nervous-system label, and of the 150 notes they file there, the dictionary found all 15 clear
seizures. But when I re-read 24 notes, I agreed with my first reading on 18 (Cohen's κ 0.48, moderate). A vet as second
reader is the next step before using this for drug safety.

## Design choice 4: adjust for case mix before ranking clinics

A clinic group wants to know which clinics have more anesthetic complications than they should. Raw rates punish
clinics that take sicker patients, and small clinics swing by chance.

The scorecards compare observed with expected complications after adjusting for ASA class, emergency status, breed
shape (flat-faced dogs carry more risk), species, procedure and age. Funnel plots show which differences are unlikely to
be chance, empirical-Bayes shrinkage keeps small clinics from topping the table by luck, and each clinic gets a rank
range rather than a single rank.

![Funnel plots of complications, deaths and dental charting by clinic, and antibiotic stewardship scores](../../assets/tailsignal/benchmarks.png)

At current volume, complication rankings match the planted quality differences (rank correlation 0.97). Death rates do
not: a clinic expects about one anesthetic death in four years, and ranking them needs roughly 5,000 procedures per
clinic. The product rule is to report complications by clinic and deaths only pooled. The same scorecards flag clinics
that under-record dental disease, which is also a revenue gap: they find a third less disease, so they book fewer
cleanings.

## Design choice 5: two lab visits, and a stress test

Chronic kidney disease is common in older cats, and diagnosing it earlier gives time for diet changes and monitoring.
Bradley et al. (2019) trained a model on 106,251 Banfield cats that became Antech's RenalTech. I wrote a specification
before running anything and compared six models on index lab panels, predicting a diagnosis 1–24 months later.

| Model (10× network) | AUC |
|---|---|
| Latest creatinine | 0.787 |
| Creatinine, BUN, urine concentration and age, one visit | 0.861 |
| Same four, two visits plus change per year | 0.894 |
| + urine protein, pH, white cells | 0.918 |
| **+ SDMA** | **0.965** |

Change over time separates early kidney decline from a cat whose creatinine runs high. SDMA, a newer
kidney marker, adds the most. Flagging cats a long way ahead stays hard: at 99% specificity the best model catches 23%
of cats diagnosed 12–24 months later, against 44% published on real Banfield data.

![Share of future cases flagged at 99% specificity by lead time, against Bradley et al. 2019](../../assets/tailsignal/kidney_lead.png)

Two follow-up checks, each written down first:

- **Stress test.** Real cats have conditions that fool kidney tests: hyperthyroidism lowers creatinine (it rose from
  1.0 to 1.5 mg/dL after treatment in [Peterson et al. 2018](https://pmc.ncbi.nlm.nih.gov/articles/PMC5787157)),
  dehydration raises it, and muscle loss lowers it. Adding all three cut the creatinine-only AUC from 0.79 to 0.73,
  while the SDMA model held at 0.96.
- **Does it pay?** A decision-curve analysis asks whether rechecking flagged cats beats rechecking every senior cat
  or none. The SDMA model wins across every trade-off from 1 to 19 unneeded rechecks per early catch, at about 1.2 to
  1.5 rechecks per cat later diagnosed.

![Net benefit of rechecking flagged cats by threshold](../../assets/tailsignal/kidney_decision.png)

## What failed, and what it taught

These failed tests changed my plans:

| Test | What happened | What it means |
|---|---|---|
| Customer segments, first design | Unstable across resamples | Rebuilt on behavior rates; the revised segments are stable |
| Demand forecast intervals | 80% ranges covered only 64% of weeks | Recalibrated from past errors (conformal): now 82%. Staffing to any upper range still cost more than staffing to the forecast |
| Early warning from daycare | The signal was engagement, not early illness | Pets active in more channels get seen more |
| Reminder targeting | Reminders help on average, but the pilot was too small to learn who | About 6,000 households to learn it; designed below |
| Drug safety in clinic records | Recovers the planted risk only at 10× the network | Rare side effects need about 130× today's network |
| Kidney: rechecks at lower prevalence | Expected to double; rose only 27% | At high specificity few healthy cats are flagged |

## The business case

I built two business cases as formula-driven workbooks. For a data business that pays partner clinics to share data,
the base case does not break even within five years: partner incentives and license volume are the levers.

The picture changes for a company that owns its clinics, such as Destination Pet. It owns the data, pays no one to
share it, and captures the operating savings itself. At 190 locations the base scenario reaches about $2.7M of net
benefit in Year 5 and pays back in Year 4. Seventy-one percent of the value is staffing savings, not data sales. Buying
data from about 100 independent vet clinics on top adds $0.4M in Year 5, all from drug-safety studies the owned network
can't reach alone. Six of the
assumptions need the company's own figures, so the case ships with an interactive version where each one is a slider.

It also enables something the simulation could not do: a randomized reminder trial. Owning the clinics means
randomizing an extra reminder across 6,000 households, stratified by clinic and engagement. That gives 80% power to
detect a 5-point difference in the reminder's effect between low- and high-engagement households, which is the
question the pilot was too small to answer.

## Scale gates

Complication scorecards and resistance tables work at today's volume. Other products unlock at a stated volume, and
those volumes set the partner-recruitment plan.

## Engineering

- **Stack:** Python with uv, DuckDB and dbt (a Snowflake target is configured), Splink for matching, scikit-learn,
  statsmodels and statsforecast for models, FastAPI for a metered data API and partner portal.
- **Data products:** versioned releases with content hashes and diffs, stored in DuckLake (Parquet fallback).
- **Tests:** 28 dbt tests for data quality and privacy; 26 unit tests run on every push with GitHub Actions.
- **Specifications:** every analysis has a pre-registered spec in `docs/`, written before its code ran, and a results
  file that scores each expectation.

## Limits

- **Partner records are simulated.** Real records are messier, and effects would be smaller and noisier.
- **The kidney model's accuracy is optimistic,** because the simulated decline was generated from the same lab values.
  It needs validation on real lab histories through a lab or clinic partner.
- **The text labels have one reader.** A second, clinical reader is needed before the dictionary supports drug-safety
  work.
- **The business case uses planning assumptions** for six figures only the company can confirm.

The code, specifications and results are at [github.com/RonCom/tailsignal](https://github.com/RonCom/tailsignal).
