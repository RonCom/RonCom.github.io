---
date: 2026-10-11
draft: true
slug: finra-enforcement-nlp
authors:
  - chris
categories:
  - Finance
  - NLP
  - Evaluation
description: Predicting which FINRA rules a disciplinary case charged from its facts, with labels read from 6,000 case documents, three classifiers scored once on 2024-2026, and a test of whether the full case document beats FINRA's own summary. Every expectation was written down before the data was pulled; three of three failed.
---

# Which FINRA rules did this conduct break? Classifiers tested against FINRA's own charges

FINRA, the regulator for U.S. broker-dealers, publishes a summary of every disciplinary action it takes each month. An enforcement or compliance team reading a complaint, a tip or an exam finding asks "**which rules does this conduct touch?**" I built classifiers that answer it from the facts of a case with the rule numbers removed, and tested them against the rules FINRA actually charged. The expectations were written down before any data was pulled, and the test years were scored once. All three classifier expectations failed, for one reason: FINRA's summaries name the conduct in the rules' own words.

<!-- more -->

!!! abstract "TL;DR"
    - **Data:** 6,198 cases from FINRA's monthly "Disciplinary and Other FINRA Actions" reports, January 2016 to September 2026 (129 months), and each case's full document (AWC, complaint or decision) from FINRA's Disciplinary Actions Online.
    - **Labels:** rules read from the violation sentences of each case document by a rule-based extractor, grouped into 15 rule series; four 100-case hand-checks (precision 0.88-0.97, recall 0.94-1.00).
    - **Models:** TF-IDF with logistic regression, ModernBERT-base fine-tuned (three seeds averaged), and a local LLM (gemma4 through Ollama) zero-shot.
    - **Result on 2024-2026 (1,143 cases):** macro-F1 **0.693 baseline, 0.694 fine-tuned, 0.664 zero-shot**. The fine-tuned model didn't beat the baseline by the 0.05 called for (H1 failed), and the zero-shot model came within 0.03 of it, not 0.10 behind (H2 failed).
    - **Full document vs. summary:** a second model reading each case's facts section scored **0.614 against the summary model's 0.681** on 300 test cases (H6 failed in the other direction).
    - **Stack:** Python, DuckDB, scikit-learn, PyTorch and Hugging Face Transformers, Ollama, PyMuPDF and Tesseract, pytest, uv. Code: [github.com/RonCom/finra-enforcement-nlp](https://github.com/RonCom/finra-enforcement-nlp).

!!! warning "Settled charges aren't every rule the conduct broke"
    The labels are the rules FINRA charged and the respondent settled or was found to have violated. Conduct can touch rules FINRA chose not to charge. The summaries are also written by FINRA staff who know the charges, so scores here are an upper bound for classifying a raw complaint or tip.

## The question and the pass lines

The specification (`docs/spec.md` in the repo) fixes the sources, the labels, the models, the split by action date (train 2016-2022, validation 2023, test 2024-2026) and six expectations:

| ID | Test | Pass line |
| --- | --- | --- |
| H1 | Fine-tuned transformer vs. TF-IDF baseline, test macro-F1 | Gain ≥ 0.05 |
| H2 | Zero-shot local model vs. fine-tuned transformer | Zero-shot lower by ≥ 0.10 |
| H3 | Retrieval over the FINRA Rulebook: a charged rule in the top 5 chunks | ≥ 70% |
| H4 | Out-of-scope questions declined | ≥ 18 of 20 |
| H5 | LLM judge vs. hand labels, Cohen's κ | ≥ 0.60 |
| H6 | Full case document vs. monthly summary as input | Gain ≥ 0.03 |

Every rule changed after data was seen is in a dated change log at the bottom of the spec, with the reason. Test scoring runs only from code at a `freeze-*` git tag on a clean tree, and refuses to run twice. H3 to H5 cover a retrieval system over the FINRA Rulebook; they're built and waiting on the Rulebook download (below).

## Cases: 129 monthly reports

FINRA's monthly reports go back to 2016 as PDFs and recent years as HTML. The scraper found all 129 published months and parsed 6,657 write-ups into 6,198 unique cases (a case can appear in more than one month). Along the way it had to handle quarterly reviews filed under the same path, a misspelled file name on five 2017 reports, and month names shortened or reordered in file names.

Each summary's numbered rule references ("FINRA Rule 3110", "Exchange Act Rule 17a-4") are replaced with `[RULE]`, and a check fails the build if any numbered reference survives. Rule titles stay in: "outside business activities" describes the conduct as much as it names the rule.

## Labels: reading the charges from the case documents

Recent summaries leave out rule numbers for most cases, so the labels come from each case's full document. The extractor reads the rules from the sentences that state a violation ("...and therefore violated FINRA Rules 3110 and 2010") and skips the procedural-rights waiver, earlier cases cited as disciplinary history, and rules quoted only as background. Where a document gives no rule, the summary's own citations are the label (20 cases), then an OCR pass over scanned or garbled PDFs with Tesseract (114 cases). 6,111 of 6,198 cases have labels.

The spec set a gate of 0.98 precision and recall on 100 hand-checked cases. Four rounds of checks scored precision 0.881, 0.907, 0.973 and 0.936 and recall 0.966, 0.944, 1.000 and 0.981. Each round's errors became extractor rules: OCR misreadings of "FINRA" and digits, footnote numbers fused to rule numbers, prior-case sentences, procedural rules. The gate was missed, and I accepted it in the change log: at the level the classifiers use, rule series, 3-5% of checked cases carry a wrong label.

Labels are rule series as the Rulebook numbers them (FINRA 1000 registration, 2000 duties and conflicts, 3000 supervision, 4000 books and records, 8000 investigations, and so on), plus MSRB rules, SEC rules, Exchange Act and Securities Act sections, and Regulations NMS and SHO. Two rules were needed before the classes made sense:

- **Rule 2010 is dropped.** FINRA charges "standards of commercial honor" next to nearly every other violation.
- **NASD rules map to their FINRA successors.** NASD 3010 became FINRA 3110 in the 2008-2012 consolidation; without the map, one obligation split into two classes by conduct date, and the NASD classes scored F1 0.24-0.53 on validation.

## Three classifiers

- **Baseline:** TF-IDF over words and word pairs, one-vs-rest logistic regression with balanced class weights.
- **Fine-tuned:** ModernBERT-base with a multi-label head, positive-class weights of √(negatives/positives) capped at 10, six epochs with the best epoch chosen on validation. One seed's macro-F1 swung by 0.028 from another's, so the test uses the average of three seeds' probabilities.
- **Zero-shot:** gemma4 (26B, through Ollama on an 8 GB GPU) gets a one-line description of each series and returns a probability per series in a fixed JSON schema. Two prompt versions were compared on validation; the second won (0.707 vs 0.672 over series with 10+ cases). Ollama's flash attention, an 8-bit key-value cache and two parallel requests made it 35% faster, after a recheck showed the answers hadn't moved.

On validation (2023), macro-F1 over series with 10+ cases was 0.782 for the baseline, 0.813 fine-tuned and 0.707 zero-shot.

## Test results, 2024-2026

| Model | Macro-F1, all 15 series | Macro-F1, 14 series with 10+ test cases |
| --- | --- | --- |
| TF-IDF + logistic regression | 0.693 | 0.714 |
| ModernBERT-base, fine-tuned, average of 3 seeds | 0.694 | 0.715 |
| gemma4:26b zero-shot | 0.664 | 0.691 |

| ID | Result | 95% interval (paired bootstrap) | Verdict |
| --- | --- | --- | --- |
| H1 | +0.000 (fine-tuned minus baseline) | -0.023 to +0.025 | Failed |
| H2 | +0.030 (fine-tuned minus zero-shot) | -0.006 to +0.073 | Failed: labeled data adds little here |

The two trained models tie overall and split by series. The fine-tuned model leads on SEC rules (0.657 vs 0.581); the baseline leads on Exchange Act sections (0.687 vs 0.612). Both score 0.95 on supervision and 8210 cooperation cases, the two biggest series.

The zero-shot model is close wherever a series has its own vocabulary: 8210 cooperation 0.959, MSRB 0.905, trade reporting 0.905. It loses where the label depends on FINRA's charging habits. It predicts registration (FINRA 1000) for 143 test cases against 56 true ones, because any mention of a Form U4 reads as a registration violation. And it splits statutes from their rules differently from FINRA's drafting.

**Why H1 and H2 failed.** Monthly summaries are short and name the conduct in the rules' own words: "failed to supervise", "outside business activity", "refused to appear for on-the-record testimony". With rule numbers masked, those phrases carry most of the signal, and word pairs capture them as well as a fine-tuned encoder does. A model with no labeled data, told only what each series covers, came within 0.03.

### Calibration

![Reliability of the three classifiers' probabilities on the test split](../../assets/finra-enforcement-nlp/reliability_test.svg)

Every case-and-series probability, binned. The fine-tuned model has the lowest calibration error (0.013) because it commits: 86% of its probabilities are under 0.1 and 7% over 0.9, where it's close to right. In its 0.5-0.6 and 0.7-0.8 bins it overstates by 0.14-0.15. The baseline's class weights push its low-middle probabilities up (of points it scores 0.1-0.2, 3.5% are true). The zero-shot model answers in steps of 0.1 and is overconfident: of the points it scores 0.8-0.9, 46% are true labels. Its F1 is close at the 0.5 threshold, but its probabilities can't rank cases for review.

## Does the full case document beat the summary?

The summary is FINRA's one paragraph about a case; the AWC behind it runs pages. H6 asked whether a model reading the document does better. The catch is that the labels come from the document's violation sentences, so the whole document hands the model its answer: masking the numbers doesn't hide "violated [RULE] by failing to supervise".

So the document input is its facts section, cleaned until it no longer states the charge. Five extraction rounds, each checked on printed samples before any training:

1. Start at the facts heading; drop "(Violation of ...)" sub-headings.
2. Drop every sentence that states a violation.
3. Drop sentences that quote or explain a rule ("[RULE] prohibits registered persons from being an employee of another person..."). Quoting the rule names the charge as plainly as its number does.
4. Stop at the sanctions, and drop sanction sentences wherever they fall. A bar or a fine of a set size points to the charge, and decisions and appeals discuss sanctions throughout.
5. Match all of the above through OCR misreadings ("conscnt to thc imposition", "viola?d").

The first version kept 2,820 of 6,125 cases, because it stopped at the first violation sentence and lost every later charge's narrative. The last kept 5,493, with 2 sentences still carrying a rule number and about 3 cases with sanction boilerplate the OCR garbled.

A second ModernBERT was trained on 4,007 facts sections the same way as the summary model and compared on 300 random test cases:

| Input | Macro-F1, all series | Series with 10+ cases in the sample |
| --- | --- | --- |
| Monthly summary | **0.681** | **0.835** |
| Facts section | 0.614 | 0.773 |

The facts model is 0.067 behind (95% interval -0.114 to -0.016; no bootstrap resample above zero). It loses most on statutes (Exchange Act sections 0.571 vs 0.731), where the sentences that separate Section 17(a) from Rule 17a-4 were the ones removed as explaining a rule. It over-predicts registration (33 predictions against 17 true), because facts sections mention Form U4 as background in many cases. Where the conduct names the series, the two tie: 8210 cooperation, supervision, MSRB, trade reporting.

This isn't a neutral comparison of documents. The facts input is what's left after removing everything that gives the label away, and those removals take some conduct with them. What it shows is that FINRA's summary carries at least as much usable signal as the stripped narrative it was written from.

## Still to come: retrieval over the Rulebook

The second half of the project asks whether a retrieval system over the FINRA Rulebook retrieves and cites the rules FINRA charged, and declines questions the Rulebook doesn't answer. It reuses the design from my [Medicare policy RAG](policy-rag.md): rule pages split into chunks of about 250 words, BM25 and embedding search (nomic-embed-text) fused by reciprocal rank, and answers from the local model that cite each rule or decline.

The code and tests are done, and these were fixed before the Rulebook was downloaded:

- **H3 scoring:** NASD rules count as their FINRA successors; Rule 2010 and procedural rules don't count.
- **Correct answer:** at least one cited rule FINRA charged, and none unrelated to the conduct.
- **Out-of-scope questions:** 20, reviewed by hand. One was dropped because FINRA Rule 2210 partly answers it.

FINRA's terms of use restrict bulk copying and machine-learning use, with an exception for fair use; this is non-commercial research, the README carries FINRA's copyright notice, and no FINRA text is redistributed beyond short excerpts. The download runs at one page every three seconds and backs off when finra.org asks it to. This section will carry H3 to H5 when they're scored.

## Engineering

- **One DuckDB file per job,** so the facts extraction, the Rulebook download and the index build run side by side.
- **A polite HTTP client** with an on-disk cache, a declared User-Agent, and backoff that doubles the gap between requests on each 429.
- **Resumable LLM runs:** every zero-shot answer is stored with its prompt version and model, so an interrupted run picks up where it stopped and a prompt change is a new version, not an overwrite.
- **Tests on every push** (GitHub Actions), including the masking check and the facts-filter cases found in each extraction round.

## Limits

- **Label noise** of 3-5% of cases at the series level is about the size of the differences between the models.
- **Summaries only** for H1 and H2; H6 tested facts sections and found them worse.
- **Rare series:** "other" (12 test cases, merged rare series) scores 0 for both trained models, which costs them about 0.05 of all-series macro-F1.
- **Validation to test drift:** both trained models lost 0.07-0.08 from 2023 to 2024-2026; the test years bring series that were rare in training (FINRA 5000 has 48 test cases against 8 in validation). The zero-shot model, with no training labels, held steady (0.667 to 0.664).
- **One zero-shot model and prompt,** series descriptions written by hand.

The code, spec with change log, and full results are at [github.com/RonCom/finra-enforcement-nlp](https://github.com/RonCom/finra-enforcement-nlp).
