---
description: TailSignal's roadmap, first 90 days, data to collect, open research items and the proposed randomized reminder trial.
---

# Roadmap and next 90 days

The order follows what works at today's scale: products that need little volume go first, because they earn data
access; studies that need a large network come later.

## First 90 days with real data

| Days | Focus | Deliverables |
|---|---|---|
| 1–30 | Audit and connect | Inventory of each practice-management and booking system, its fields and who owns which identifiers; privacy and release rules agreed; household-then-pet linkage running on real records |
| 31–60 | Ship first products | Clinic complication, dental-charting and stewardship scorecards for 2–3 clinic groups; regional antibiogram in the portal; monthly data-quality metrics (match rates, completeness, freshness) |
| 61–90 | Prove value | One paid insight product piloted; staffing-forecast pilot in 2–3 locations against a matched control; partner targets set from the scale gates; a model-validation standard agreed |

## What each product needs before it can be sold

| Product | Gate | Status |
|---|---|---|
| Clinic scorecards, antibiograms, resistance feed, drug-safety landscape | None beyond real data | Ready |
| Feline kidney risk flag | Validation on real lab histories; a lab or clinic partner | Pilot |
| Staffing forecasts | Live pilot confirms the 7% saving | Pilot |
| Data licenses and Pet Health Index | ~100 clinics, so privacy suppression falls | Gated |
| Common side-effect studies | ~80× a nine-clinic network | Gated |
| Rare side-effect studies | ~130× a nine-clinic network; likely a consortium | Gated |
| Targeted reminders | Randomized trial below | Trial |

## Proposed randomized reminder trial

The pilot (2,927 memberships) showed reminders cut 180-day lapse from 14.1% to 12.4% on average, but was too small to
learn who benefits. A company that owns its clinics can run the trial that settles it.

| | Pre-registered design |
|---|---|
| Question | Does an extra personal reminder (text, then a call) reduce lapsed wellness care, and does the effect differ between low- and high-engagement households? |
| Randomization | Household, 1:1, stratified by clinic and engagement segment (defined from the prior 12 months, frozen before launch) |
| Primary outcome | No wellness visit within 180 days of the due date |
| Primary hypothesis | The reminder's effect differs by at least 5 percentage points between segments |
| Sample size | 6,000 households (1,500 per arm per segment): 80% power, two-sided 5% |
| Average effect | 49% power at 6,000; about 12,500 households for 80% power, decided before launch |
| Analysis | Risk difference by arm within segment, adjusted for clinic; interaction test; fixed horizon |
| Decision rule | Effect differs: target by segment. No difference: remind everyone if the average effect beats the cost per contact |

To confirm with the company: monthly wellness due dates (sets enrollment time), the actual 180-day lapse rate, and
the cost per personal reminder.

## Open research items

| Item | Why | Status |
|---|---|---|
| **Exposure denominators for Model A** | Spontaneous reports give proportional ratios, not rates. Model A2 already computes true rates using clinic prescriptions as the denominator, but only on simulated records. Public dose or sales data for companion-animal products is not available, so the next step is a reporting rate per 10,000 prescriptions from partner clinics, linking FDA report counts to real dispensing volume | **Not yet done** |
| Clinical second reader for clinic-note labels | Self-agreement κ 0.48 on 24 notes; a vet or vet nurse would make it a measured two-reader result | Planned |
| Model B: breed-condition risk with partial pooling (H4) | Needs Dog Aging Project data | Awaiting access |
| Kidney model on real lab histories | Simulated accuracy is optimistic | Needs a lab or clinic partner |
| Confounders correlated with disease | Treating hyperthyroidism often reveals hidden kidney disease; the stress test applied confounders independently | Planned |
| Staffing decision rule at handler level | A newsvendor rule should account for whole-handler rounding | Planned |

## Data to collect

| Data | Unlocks | Source |
|---|---|---|
| Owner identifiers in every data-sharing agreement | Accurate linkage across clinics and services | Data agreements |
| Prescriptions and dispensing volume | Rates for drug safety (Model A), adherence analytics | Partner clinics, pharmacy partners |
| Repeat lab panels with SDMA for senior cats | Kidney model validation | Clinics, lab partner |
| Daycare staffing and turned-away days | Staffing pilot; the business case's largest line | Operations |
| Wellness due dates, lapse rates, reminder costs | Reminder trial | Operations, CRM |
| Dog Aging Project | Model B | Research data request |
