---
description: TailSignal's products, buyers, pricing assumptions, scale gates and go-to-market sequence.
---

# Products and go-to-market

TailSignal sells four kinds of things from the same linked data: the data itself, packaged insight, scores from
models, and analytics inside tools partners already use. In order: embedded analytics earn data access,
research reports bring the earliest outside revenue, and data licenses and studies unlock only at stated network
sizes. Prices are **planning assumptions** for the business case, to be replaced with buyer quotes.

## The four revenue models

| | Data products | Research and insights | Commercial models | Embedded analytics |
|---|---|---|---|---|
| What is sold | Curated, de-identified data | Packaged analysis and benchmarks | Scores and forecasts, not the data | Features inside a tool partners already use |
| TailSignal offering | Pet Health Data Feed: cohort and aggregate tables, resistance feed | Pet Health Index, oral health report, drug-safety landscape | Scoring API: drug signals, demand forecasts, kidney risk | Partner portal: scorecards, antibiograms, staffing forecasts |
| Primary buyers | Animal-health pharma, pet food, insurers, researchers | Pet brands, investors and PE diligence, trade bodies | Insurers, pharma, multi-site operators | The clinics and operators who contribute data |
| Delivery | Parquet files, Snowflake share, REST API | Reports, dashboards, briefings | REST scoring API, batch files | Portal pages and in-workflow alerts |
| Pricing model | Annual license by scope and refresh | Subscription or one-off study | Per call, per score, or platform license | Per-location add-on; free tier in exchange for data |
| Planning price | $120,000 a year per license | $40,000 per report | $80,000 a year per client | $1,800 per clinic per year (premium tier) |
| Built in the repo | `product_pet_cohort`, `product_condition_prevalence`, releases | Prevalence mart, Models A, AMR, oral health | Models A, A2, forecasting, feline kidney; FastAPI | Clinic benchmarks, AMR, portal route |

## Product catalog: what can be sold now

| Product | Buyer | Sellable now? | What gates it | Evidence |
|---|---|---|---|---|
| Clinic complication, dental-charting and stewardship scorecards | Clinic groups, acquirers, insurers | **Yes** | Works at ~555 anesthetics per clinic over 4 years | Rank correlation 0.97 with true quality |
| Regional antibiograms ("which antibiotic still works") | Clinics | **Yes** | Public data; refreshed yearly | 26,396 real FDA isolates |
| MRSP and multidrug-resistance trend feed | Public health, One Health programs, antibiotic and diagnostics makers | **Yes** | Public data | MRSP 31% → 43% in skin isolates |
| Drug-safety landscape by breed | Animal-health manufacturers, regulators | **Yes** | Public data | 0 false alarms per 1,000; MDR1 breed risk ranked 16–30 |
| Time-to-onset evidence for study design | Manufacturers | **Yes** | Public data | Isoxazoline seizures: median 2 days, 89% within 50 days |
| Outcome dictionaries and annotation services | Manufacturers, clinic software vendors | **Yes, with clinical review** | A clinical second reader | 30% precision on real notes before review, 15 of 15 clear seizures found |
| Oral health insights report | Pet food and dental-care makers | **Yes** (format proven on simulated data) | Real partner data | Toy breeds 2.5× the odds; 46% of diagnosed dogs cleaned the same year |
| Feline kidney risk flag on lab reports | Lab-diagnostics partners, renal-diet makers, clinic groups | **Pilot** | Validation on real lab histories | AUC 0.965; ~1.2–1.5 rechecks per cat later diagnosed |
| Daycare staffing forecasts | Multi-site operators | **Pilot** | Confirm the 7% saving in live locations | Backtest: staffing cost 7% below baseline |
| Pet Health Index and data licenses | Insurers, pharma, pet food | **At ~100 clinics** | Privacy suppression: only 39% of index cells releasable today | k = 10 enforced in every release |
| Common side-effect validation studies | Manufacturers | **At ~80× today's network** (~720 clinics) | Event counts | Model A2 power analysis |
| Rare side-effect validation studies | Manufacturers, regulators | **At ~130× today's network** (~1,170 clinics) | Event counts; likely a consortium | Model A2 power analysis |
| Targeted reminders ("who to remind") | Clinics, wellness-plan providers | **Not yet** | A randomized trial of ~6,000 households | Pilot too small to learn who benefits |
| Cross-channel illness early warning | Insurers | **Not as a headline product** | Lift was engagement, not warning | +0.007 AUC; +0.009–0.022 for pets active in other channels |

## Go-to-market sequence

1. **Give clinics something back first.** Free complication, dental and stewardship scorecards plus the regional
   antibiogram in the partner portal, given to clinic groups in exchange for their data; the premium tier adds group
   benchmarking at $1,800 per clinic per year.
2. **Sell insight while the network is small.** Research reports (oral health, resistance trends, drug-safety
   landscape) need little network volume because they lean on public data or aggregate cuts. Planning volume: 2 reports
   in Year 1, 10 in Year 5.
3. **Open data licenses at about 100 clinics.** Below that, privacy suppression removes too many cells (8% of pets at
   9 clinics; 61% of Pet Health Index cells). Planning: 2 licensees in Year 2, 8 in Year 5.
4. **Scoring API once models are validated.** Drug-signal and demand-forecast scoring: 1 client in Year 2, 7 in
   Year 5. Each needs a model card, calibration report, drift monitoring and a stated scope of valid use.
5. **Validation studies last.** $250,000 each, unlocking at about 720 clinics (common side effects) and 1,170 (rare).
   In the partner-network Base case they are 29% of Year-5 revenue.

## Sales motions by buyer

| Buyer | Opening offer | Proof point to bring | Expansion |
|---|---|---|---|
| Clinic groups | Free quality and stewardship scorecards | Funnel plots that separate real differences from chance; rank ranges for each clinic | Premium benchmarking; staffing forecasts; kidney flag |
| PE diligence teams | Target-vs-network benchmark from 2–4 years of records | Complication and charting gaps measurable at single-clinic volume | Portfolio-wide quality monitoring |
| Animal-health manufacturers | Breed-level drug-safety landscape for their products | 0 false alarms vs 43 per 1,000; MDR1 breed control found | Time-to-onset analyses; validation studies at scale |
| Pet food and dental makers | Oral health report | Toy-breed risk, treatment gap | Pet Health Index license |
| Insurers | Pet Health Index by breed, region, condition | Shrunken estimates with intervals: no repricing on noise | Data license; network quality tiers |
| Diagnostics companies | Resistance trends and testing gaps by region | Rising MRSP makes the case for culture and susceptibility testing | Kidney flag co-development |
| Public health | Resistance trend feed | Artifacts found and corrected (site breakpoints, selective testing) | Syndromic surveillance feed |

## Pricing and packaging (planning assumptions)

| Package | Price | Basis |
|---|---|---|
| Partner scorecard, free tier | $0, in exchange for data | Partner acquisition |
| Premium scorecard | $1,800 per clinic per year; 40% uptake assumed | Group benchmarking beyond the free tier |
| Research report | $40,000 average | Comparable to custom market studies |
| Data license | $120,000 a year | De-identified cohort, Pet Health Index, resistance feed |
| Scoring/API contract | $80,000 a year per client | Drug-signal and forecast scoring |
| Signal-validation study | $250,000 per study | EHR cohort study for a manufacturer or regulator |

In the partner-network business case, at 1,500 clinics in Year 5, revenue is $4.25M and
each clinic brings about $2,800 of revenue against a $1,000 incentive. See [Business case](business-case.md).

## How the API enforces the commercial model

Each customer key has a role, and each role has entitlements. Every successful call is logged with key, endpoint, row
count and release version, which is the basis for usage billing.

| Role | Can read |
|---|---|
| Insurer | Pet Health Index |
| Pharma | Drug-safety signals, Pet Health Index |
| Public health | Antibiogram, drug-safety signals |
| Clinic | Its own scorecard and portal page, antibiogram |
| Admin | Everything |

A clinic key cannot read another clinic's scorecard; unknown keys are rejected. Both are covered by automated tests. Try it as a customer in the [customer portal](portal.md).

## Results dashboard

A partner or buyer sees clinic scorecards (with a switch to the 10× network),
antibiotic-resistance trends and the regional antibiogram, kidney early warning, and the products-by-scale table.
[Open it full screen](interactive/tailsignal_dashboard.html).

<iframe src="../interactive/tailsignal_dashboard.html" title="TailSignal results dashboard" style="width:100%;height:900px;border:1px solid var(--md-default-fg-color--lightest);border-radius:8px" loading="lazy"></iframe>

## Risks to the commercial plan

| Risk | Mitigation |
|---|---|
| Partners object to resale of their data | Revenue share and free analytics; insight and scores rather than raw rows where possible |
| Re-identification | k = 10 enforced by tests that fail the build; generalized geography and dates; no owner identifiers in products |
| Buyers commoditize raw data | Keep the highest-value signals in models and scores |
| Model liability and drift | Model cards, calibration reports, drift monitoring, contractual scope |
| Free portal undervalued | Tiering: free benchmarks, paid predictive features |
| Scale gates arrive later than planned | Studies are 29% of Year-5 Base revenue; without them Year-5 revenue is $3.0M |
