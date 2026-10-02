---
description: Executive summary of TailSignal, a pet health data platform that links vet, daycare, grooming and wellness-plan records and turns them into products.
---

# TailSignal: executive summary

TailSignal is a working model of a pet health data business. It links each pet's records across the businesses that
care for it (vet clinics, daycare and boarding, grooming, wellness plans), standardizes and protects them, and turns
the result into four kinds of revenue. Every analysis had a written test before it ran, and the results below include
the ones that failed.

!!! info "How to read this section"
    | If you are | Start with |
    |---|---|
    | CEO | This page, then [Business case](business-case.md) |
    | Head of sales | [Products and go-to-market](commercial.md), then the [customer portal](portal.md) |
    | CTO | [Platform and engineering](platform.md) |
    | Head of research | [Research paper](paper.md): full methods, results and references |
    | Everyone | [Roadmap and next 90 days](roadmap.md) |

    Interactive pages: the [customer portal](portal.md) (sign in as an insurer, manufacturer, agency or clinic and use
    the data products), the [business-case explorer](business-case.md#interactive-explorer) (adjust the assumptions and
    watch EBITDA by year) and the [results dashboard](commercial.md#results-dashboard).

    Code, specifications and full results: [github.com/RonCom/tailsignal](https://github.com/RonCom/tailsignal).
    A shorter narrative version is on the [blog](../../blog/posts/tailsignal.md).

## The opportunity

A dog is seen at the vet for illness, at daycare for behavior and attendance, by a groomer for skin, ears and lumps,
and by a wellness plan for loyalty. Each business holds one slice, none shares an ID, and no buyer can see the whole
pet. Buyers for the linked record include animal-health manufacturers, pet food and dental
makers, insurers, diagnostics companies, public-health agencies and the clinic groups that produce the data.

Corporate groups own about 30% of the roughly 34,000 US veterinary practices and over half of companion-animal revenue
(AVMA, via [dvm360](https://www.dvm360.com/view/2025-economic-state-of-the-veterinary-profession-trends-and-opportunities-for-your-practice)).
US pet industry spending reached $158 billion in 2025 across 95 million pet-owning households
([APPA](https://americanpetproducts.org/news/u.s.-pet-industry-reaches-158-billion-in-2025-poised-for-continued-growth-in-2026)).

## What was built

| Stage | What exists | Scale |
|---|---|---|
| Collect | 6 simulated partner systems in CSV, JSON-lines and pipe-delimited files; ingest for FDA, Census, city and clinical-text data | 26,099 pet records; 1.36M real FDA reports |
| Organize | dbt models, breed and diagnosis taxonomies, two-stage record linkage, k-anonymous products | 12,666 pets; match F1 0.983 vs 0.904 |
| Analyze | Pre-registered analyses: drug safety, antibiotic resistance, clinic quality, feline kidney disease, oral health, segments, forecasting, early warning, reminders | Each scored against written expectations; misses reported |
| Commercialize | Pet Health Index, versioned data releases, metered API with entitlements, partner portal, two business cases | 7 products per release; 5 of 5 release checks pass |

## Headline results

| Result | Data | What it means commercially |
|---|---|---|
| Breed-aware drug-safety model: 0 false alarms per 1,000 vs 43 for the standard method; finds the herding-breed ivermectin risk at rank 16–30 vs 235–1,545 | 970,167 real FDA dog reports | A drug-safety landscape manufacturers can act on without drowning in false alerts |
| Methicillin-resistant staph in dog skin infections rose from 31% to 43%, 2017–2024 | 26,396 real FDA NARMS isolates | Regional "which antibiotic still works" tables for clinics; a trend feed for public health |
| Clinic complication scorecards rank correctly at today's volume (rank correlation 0.97) | Simulated clinic records | A product for clinic groups and acquirers now; death-rate scorecards need ~5,000 procedures per clinic |
| Feline kidney early warning: AUC 0.965 with two lab visits and SDMA; holds at 0.96 under real-world confounders | Simulated lab panels, calibrated to IRIS and published studies | A risk flag on each senior cat's lab report; costs ~1.2–1.5 rechecks per cat later diagnosed |
| Seizure text-mining fell from 75% to 30% precision on real UK clinic notes, but found every clear seizure (15 of 15) | 4,999 real clinic notes (PetEVAL) | Outcome dictionaries need clinical review; sellable as an annotation service |
| Drug-safety studies in clinic records recover the planted risk only at 10× the network; rare events need ~130× | Simulated clinic records | Rare-event studies need a consortium; common-event studies need ~80× |

## What it is worth

Two cases, both formula-driven workbooks with Low / Base / High scenarios and every assumption editable.

| | Partner network (pays clinics to share data) | Owned network (a company that owns its clinics) |
|---|---|---|
| Year-5 result, Base | Revenue $4.25M; EBITDA −$0.89M | Net EBITDA impact +$2.80M |
| Payback | Not within 5 years in any scenario | Year 3 (Base), Year 2 (High) |
| Largest value line | Embedded analytics (25% of revenue) and data licenses (23%) | Staffing savings from demand forecasting (70% of value) |
| Largest cost | Partner incentives ($1,000 per clinic per year) and team | Data team and system integration across acquired brands |
| Enterprise value at 12× | n/a | $33.6M (Base); $3.5M–$57.5M across scenarios |

The difference is ownership: an operator that owns every location owns its data, pays no partner incentives and
captures operating savings directly. Details and the assumptions behind each line are on
[Business case](business-case.md).

## What the results say not to sell yet

- **"Daycare data predicts illness."** The lift was engagement (pets seen more get diagnosed more), not early warning.
- **Per-clinic death rates.** At current volume they are noise; a noisy league table damages trust with vets.
- **Targeted reminders.** Reminders help on average, but the pilot was too small to learn who benefits. A randomized
  trial of 6,000 households is designed to answer it.
- **Rare side-effect studies.** They need about 130× today's clinic volume, which takes a consortium.

## Decisions for leadership

1. **Which network model applies.** Owned clinics make the case positive by Year 3; a partner network needs cheaper
   data access (free scorecards rather than cash incentives) or more data licenses to break even.
2. **First product.** Clinic complication and stewardship scorecards work at today's volume and give clinics something
   back, which earns data access.
3. **Owner identifiers in every data agreement.** Matching on pet details alone collapsed to 4% precision at scale;
   owner identifiers are what make linkage work.
4. **A staffing pilot before rollout.** Staffing savings carry 70% of the owned-network value and come from simulated
   data, so they need confirming in two or three locations first.

## What is real and what is simulated

Partner records are simulated from a fixed seed, because clinic records are not available to a portfolio project and
a known answer lets every method be checked. The simulator was calibrated to published veterinary studies and passes
26 of 35 validation checks, with every miss listed. Drug safety, antibiotic resistance and clinical text use real
public data only. Absolute accuracy on simulated data is optimistic; the comparisons between methods are the result.
