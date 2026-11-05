---
date:
  created: 2026-10-01
  updated: 2026-11-04
slug: policy-rag
authors:
  - chris
categories:
  - Healthcare
  - Generative AI
  - Evaluation
description: A local retrieval-augmented system that answers Medicare billing-policy questions with section citations, and how its retrieval, answers and LLM judge were evaluated.
---

# Answering Medicare billing-policy questions with a local RAG system

A payment-integrity reviewer who sees a flag such as "timed therapy units averaging under 15 minutes" needs the
policy that applies, with a citation they can check. I built a small retrieval-augmented generation (RAG) system that
answers billing-policy questions from four CMS manuals using local models, cites a manual section for every claim, and
declines when the manuals don't cover the question. Then I measured it: retrieval, answers, declines, and the LLM judge
itself.

<!-- more -->

!!! note "Updated November 4, 2026"
    A rerun added a cross-encoder reranker (tested twice, kept out), per-answer latency, token and cost tracing,
    and a change to how answers are scored. The judge had been passing every decline on a question the manuals do
    answer, which put the model without documents at 53%. Scored with that rule in code, it gets 1%, and agreement
    with my hand labels rose from κ 0.68 to 0.91. The numbers below are from the rerun.

!!! abstract "TL;DR"
    - **Corpus:** four CMS manuals (NCCI Policy Manual chapters 1 and 11, Medicare Benefit Policy Manual chapter 15, Claims Processing Manual chapter 5), split into 851 section-aware chunks.
    - **Stack:** Ollama (`nomic-embed-text`, `gemma4:e4b`, `gemma4:26b` as judge), DuckDB, BM25 + embeddings fused with reciprocal rank fusion, a cross-encoder reranker under test, MLflow. Everything runs on a laptop with an 8 GB GPU.
    - **Answers:** 80% correct with retrieval vs 1% for the same model without documents, scored by the judge plus a code rule that fails a decline on a question the manuals answer; 12 of 12 out-of-scope questions declined.
    - **Retrieval:** keyword search found the right section for 100% of questions written in the manuals' wording but only 40% of plain-language ones; embeddings reached 70% on those.
    - **Weak spot:** plain-language questions score 40% correct vs 91% on the manuals' wording and 88% paraphrased. That sets what to tune and what to monitor.
    - **Reranking:** a cross-encoder added 2 points at top 5, inside its interval, and no gain on answers; both tests had their rules written first, and hybrid stays.
    - **Judge:** checked against 37 hand labels: κ 0.68 for the judge alone, 0.91 with the decline rule in code.
    - **MLOps:** MLflow traces every answer with latency, tokens and cost (p95 13.2 s); a chunking sweep runs with a rule fixed in advance; Evidently watches label-free retrieval signals for drift and feeds a review queue.
    - **Code:** [github.com/RonCom/policy-rag](https://github.com/RonCom/policy-rag).

## Why this problem

My [Medicare FWA project](medicare-fwa.md) flags providers whose billing is unusual: units per patient-day near the
Medically Unlikely Edit, timed therapy units well above peers. The next question a reviewer asks is what the rule
actually says. The answers are in long, dense manuals, and a general chatbot will answer confidently without saying
where the answer came from. For review work, an answer without a source is not usable, and an invented one is worse.

So the requirements were narrow:

1. Answer only from the manuals.
2. Cite the section for every claim.
3. Say "not in these documents" when they don't cover the question.
4. Measure all three, rather than judging from a few good-looking examples.

## Design choice 1: chunk by the manuals' own structure

The manuals have their own sections: NCCI chapters use lettered sections ("V. Medically Unlikely Edits"), the CMS
manuals use numbered ones ("§20.2 Reporting of Service Units With HCPCS"). Chunks follow those sections, in pieces of
about 300 words with 40 words of overlap, so a citation points to a section a reviewer can look up.

Getting the sections right took more care than expected. The manuals open with a table of contents, body text quotes
other sections ("see 220.2 - Reasonable and Necessary Outpatient Rehabilitation Therapy Services)"), and one chapter
lists a run of section titles in the middle of a paragraph. My first version treated those as headings, and one
section swallowed 140 chunks. The fix: a line counts as a heading only if it is one of the next few entries in the
manual's own table of contents. Running headers, page labels and the revision history at the end are dropped. The
result is 851 chunks across 304 sections.

## Design choice 2: two kinds of search, combined

Billing policy is full of exact tokens: CPT codes, "modifier 59", "8 minutes", section numbers. Keyword search (BM25)
matches those well. People don't always ask in the manuals' words, though, and embeddings handle paraphrase. The
system runs both and combines the rankings with reciprocal rank fusion (hybrid search). Vectors are stored in DuckDB.

## Design choice 3: constrain the answer

The answering model sees five numbered excerpts and is told to answer only from them, cite each claim as `[n]`, and
reply `NOT IN DOCUMENTS` otherwise. A typical answer:

> **Q:** How many 15-minute units should be billed for 47 minutes of timed therapy (24 minutes of 97112 and 23 of 97110)?
>
> **A:** The total timed code treatment time of 47 minutes falls within the range for 3 units (38 to 52 minutes) [1].
> The correct coding is 2 units of code 97112 and one unit of code 97110, assigning more timed units to the service
> that took the most time [1].
>
> *[1] Claims Processing Manual Ch. 5, §20.2*

## Design choice 4: a test set that can fail

The evaluation has 62 questions. Fifty are answerable, each with a reference answer, its key facts, and the section
that contains them, checked against the manual text. Twelve ask about things the manuals don't
cover (Part D penalties, MS-DRG weights, MIPS thresholds, Medicare Advantage risk adjustment), and I confirmed each
topic is absent from the corpus.

My first 40 answerable questions were drafted from the manual text, and that turned out to matter. Keyword search
scored 100% on them, because the questions reused the manuals' wording. So I added 10 questions in the way a reviewer
might actually ask:

- "A PT clinic billed four quarter-hour blocks for about 50 minutes of hands-on work. Is that right?"
- "Does Medicare stop paying for therapy once the patient stops getting better?"
- "Can a chiropractor get paid for an X-ray they take to show a spinal problem?"

## Results: retrieval

A hit means a chunk from the gold section is among the top results.

| Method | Top 1 | Top 5 | Top 5, manual wording (40) | Top 5, plain language (10) |
|---|---|---|---|---|
| Keyword (BM25) | 76% | 88% | 100% | 40% |
| Embeddings | 70% | 90% | 95% | **70%** |
| **Hybrid** | **76%** | **92%** | 100% | 60% |
| Rerank (cross-encoder over hybrid's top 20) | 72% | 94% | 100% | 70% |

The plain-language questions changed the picture. Keyword search, perfect on manual wording, found the right
section for only 4 of 10 everyday questions; embeddings found 7. Hybrid keeps keyword search's precision on exact
terms and most of the embedding gain, which is why it answers questions here. With 10 plain questions, though, the
ranking among methods is suggestive, not settled.

![Recall at 5 by question wording and retriever](../../assets/policy-rag/retrieval_by_style.png)

Keyword search and embeddings each score the question and a chunk separately. A cross-encoder reads the two
together, which is the check plain-language misses call for, but it's too slow to run over 851 chunks, so it
re-scores hybrid's top 20. It found one more question in the top 5 (94% vs. 92%, 95% interval −4 to +8 points)
and moved two correct sections out of first place. My rule, written before the run, switched only if the interval
cleared zero, so hybrid stayed.

Recall isn't answer quality, so rerank got a second test on answers, again with the rule written first: switch only
if correctness on the 50 answerable questions rises with an interval above zero and false declines don't. Both
scored 84% on the judge's grades (interval −9 to +9 points). Rerank changed the grade on 11 questions, six better
and five worse: it fixed a question about billing a 5-minute service and declined two that hybrid answered. Hybrid
stays the answer retriever.

**What this changes.** Hybrid stays the default; dense and rerank lead on plain language by one question in ten,
which is not evidence. Everyday wording is the failure mode, so it is what monitoring watches (below) and where
the test set needs more questions.

## Results: answers

| | With retrieval | Same model, no documents |
|---|---|---|
| Correct, answerable questions (judge + decline rule) | **80%** | 1% |
| Judge alone, answerable questions | 84% | 53% |
| Declined the 12 out-of-scope questions | **12 / 12** | 9 / 12 |
| Declined an answerable question | 3 / 50 (one in prose) | – |
| Cited a gold section when answering | 89% | – |

Retrieval raises correctness from 1% to 80%, and every claim is tied to a section. The judge alone put the model
without documents at 53%, and the gap is the subject of the judge section below. Without the documents, the model
answered three out-of-scope questions (Part D penalties, Medicare Advantage risk adjustment, ACO benchmarks) from
general knowledge. Those answers were broadly reasonable, but nothing in them could be checked against a source, which
is the property a reviewer needs.

Breaking the results down by how a question is worded tells a sharper story than the averages.

![Answer grades split by whether retrieval found the gold section](../../assets/policy-rag/answer_outcomes.png)

Fourteen of the 50 answerable questions fall short of fully correct. Four are retrieval misses, all on
plain-language questions, and all four ended wrong: two declines, one of them written as prose instead of the exact
reply, and two answers from the wrong section. Asked whether Medicare stops paying once a patient stops improving,
the system retrieved a section on notifying beneficiaries instead of §220.2's rule on maintenance therapy, and
answered "yes"; the manual's answer is "not necessarily." The other ten had the right section and still fell
short: eight left out a secondary fact (the 14-day signature rule for verbal orders, the 30-day cancellation
notice, the 53-minute threshold for a fourth unit), one declined, and one said 15 minutes where the rule is 8.
Plain-language questions score 40%; manual wording 91% and paraphrased 88%.

**What this changes.** A miss became a wrong answer every time, so retrieval is the first thing to tune. But more
points are lost with the right section in hand, so the answer step is next. My working hypothesis is that the
1-4 sentence limit in the prompt drops secondary facts; that is a prompt change to test with the judge, tracked
by prompt hash.

![With and without the manuals, by question wording, and on out-of-scope questions](../../assets/policy-rag/rag_vs_closed_book.png)

Retrieval lifts manual-wording questions from 0% to 91%, paraphrased ones from 0% to 88% and plain-language ones
from 5% to 40%, and keeps every out-of-scope question declined. Without the manuals, the model declined 26 of the 50
answerable questions and got 23 of its 24 answers wrong. Plain language is where retrieval is weakest, and its
failure there is the worse kind: a wrong answer from the wrong section arrives *with a citation*, which looks more
trustworthy than an unsourced guess.

**What this changes.** Everyday questions need better retrieval before their answers can be trusted. A guard to
test later: when the monitoring signals say retrieval is unsure, decline or flag instead of answering.

## Results: can the judge be trusted?

Correctness and faithfulness were graded by a larger local model (`gemma4:26b`) than the one answering. An LLM judge
is only useful if it agrees with a person, so I labeled 37 answers by hand, choosing a mix of errors, partial answers
and declines, and compared:

| Correctness | Labels | Agreement | Cohen's κ | Scored pass rate | My pass rate |
|---|---|---|---|---|---|
| Judge alone, first run | 37 | 92% (34 / 37) | 0.68 | 86% | 84% |
| Judge + decline rule, first run | 37 | 97% (36 / 37) | **0.91** | 81% | 84% |
| Judge alone, rerun | 20 | 95% (19 / 20) | 0.77 | 90% | 85% |
| Judge + decline rule, rerun | 20 | 100% (20 / 20) | 1.00 | 85% | 85% |

The disagreements showed a pattern. The judge passed "NOT IN DOCUMENTS" replies whenever the excerpts it was shown
lacked the answer, even when the manuals contain it. It grades against what was retrieved, so a retrieval miss looks
like a correct decline. In the rerun, without documents, the model declined 26 of 50 answerable questions and the
judge passed all 26. Of the 24 answers it gave, the judge failed 23 and gave one partial credit, so 52 of its 53
points were credit for declines.

Whether a question is answerable is fixed by the test set, so that rule doesn't need a judge. A decline on an
answerable question now scores 0 in code, whether it's the exact reply or written as prose ("the excerpts do not
provide a specific answer"), and an exact decline counts as faithful, which the judge's rubric already said and the
judge once ignored. With the rule, agreement with my labels went from κ 0.68 to 0.91. The rerun's κ 1.00 is on 20
labels, the ones whose answers came out word for word the same, so it's a small sample.

![LLM judge vs human grades on the labeled answers that match the rerun](../../assets/policy-rag/judge_calibration.png)

**What this changes.** The judge grades what code can't: correctness of an answer against the reference's key facts.
Rules the test set decides exactly run in code after it, and both grades are kept, so every comparison shows the
judge alone and the judge with the rules. The calibration holds for one judge model and one judge prompt; changing
either means labeling a fresh sample.

Defining the labels mattered too. On my first pass I scored two wrong answers as "unfaithful" because they were
wrong, but they accurately quoted the (wrong) sections they were given. Faithful means supported by the excerpts, not
true; correctness is scored separately. Writing that rule down before labeling is the difference between a measured
judge and an impression of one.

## Feature engineering, and what each feature is worth

In a RAG system the features are how documents and queries are represented for search. I made each choice for a
reason, then tested it the plain way: remove one feature at a time and re-score retrieval on the same 50
questions.

![Retrieval recall with one feature removed at a time](../../assets/policy-rag/ablation.png)

| Feature | Why it's there | Recall at 5 without it |
|---|---|---|
| Table-of-contents check on headings | Cross-references look like headings | **80%** (from 88%); 4 questions lost, none gained |
| Section-aware chunks | Citations must point to one section | 90%, but a third of section-blind windows span two or more sections and can't be cited to one |
| Section heading in the chunk | Names the topic in the manual's terms | Keyword 88% (unchanged); embedding 92% (from 90%), but top 1 falls from 70% to 58% |
| Tokenizer keeping "220.2", "g-codes" | Section numbers and codes are exact evidence | 88% (top 10: 96% to 92%) |
| Stopword removal | Common words carry no topic | 88% |
| Embedding task prefixes | The model was trained with them | 92% (from 90%); slightly *better* without them |

The one feature that clearly earned its place is structural: without the table-of-contents check, the parser
finds 311 sections instead of 304, and four questions lose their gold section. The token-level choices didn't
move top-5 recall on this test set. They stay as sensible defaults, but they aren't where the next gain is.

Two embedding-side results ran against the documentation. The section heading barely matters at top 5, but it is
what puts the right section first (70% vs 58% at rank 1), and that is what hybrid fusion rewards, so it stays.
The task prefixes that `nomic-embed-text` recommends made retrieval slightly *worse*: one question better
without them, with an interval that touches zero. That isn't enough to switch on, but it is the cheapest change
to test in the next full eval.

**What this changes.** The weak spot is plain-language questions, and that is a query-side problem: a reviewer
asks about a doctor "signing off" on a therapy plan, while the manual says "certification of the plan of care."
The next feature to try is a small glossary that maps everyday terms to the manuals' vocabulary before keyword
search. It gets the same treatment: an ablation on plain-language questions, once there are more than ten of them.

## Tracking and monitoring: MLflow and Evidently

An evaluation is a snapshot. Two questions remain once the system is in use: which configuration is running and
why, and whether it still works on the questions people actually ask. I used MLflow for the first and Evidently for
the second, and the design choices were mostly about keeping both trustworthy at this scale.

**Every answer is a trace.** Each question is logged as an MLflow trace with three spans: `rag`, `retrieve` (the
five excerpts with section, page, rank and score) and `generate` (prompt version and reply). Three kinds of scores
are attached to the same trace: a deterministic check (did retrieval return the gold section?), the LLM judge's
grade with its reason, and my hand label. When the judge and I disagree, I open one trace and see what the model
was shown; that is where the judge's leniency on declines after a retrieval miss becomes obvious.

**Runs carry their lineage.** Each eval run logs the models and chunking settings, plus hashes of the question file
and both prompts, the git commit, the Ollama version and each model's digest. The question set grew from 52 to 62
during this project; without the hash, two accuracy numbers from different question sets look comparable when they
aren't. The version and digests came later, for a reason: a rerun with the same prompt hashes, still at
temperature 0, changed 35 of 62 answers. The closed-book answers changed too, and they involve no retrieval, which
points at the model or the server; the first run logged neither, so the cause can't be pinned down. Hand labels and
judge grades only count against the exact answer they graded, so a script copies grades to answers whose text came
out identical and sends only the changed ones back to the judge.

**Every answer carries its cost.** The trace records latency per step, tokens in and out, Ollama's own timings
(model load, prompt processing, generation) and dollars from a price table, which is empty for local models and
gets a row for any hosted one. On the rerun: p50 10.3 s, p95 13.2 s, with generation 11.1 s of the p95 and
retrieval 2.1 s. That points the first latency fix at the answer model. A change to the prompt, top-k or reranker
moves accuracy and cost together, and each run reports both.

**The chunk size gets tested, not defended.** I had picked 300-word chunks by judgment. `policy-rag sweep` tries
five chunkings with all three retrievers as nested MLflow runs. It sweeps retrieval only, because retrieval is
deterministic and cheap while answer grading takes hours on an 8 GB GPU, and a retrieval miss became a wrong
answer every time. The selection rule is in the config before the run:

1. the top five excerpts must fit a 3,000-word budget, since bigger chunks raise recall partly by showing the
   model more text;
2. rank by recall at 5, then plain-language recall, then MRR;
3. change the setting only if the bootstrap interval for the gain is above zero.

With 50 questions, one question is two points, so the third rule does most of the work.

**Monitoring without labels.** Nobody grades answers in use, so accuracy can't be watched directly. What can be
measured on every query is how it looks and how confident retrieval is:

- the share of query words that never appear in the manuals;
- the best keyword and embedding scores;
- whether keyword and embedding search agree on the top five;
- how scattered the top five are across sections.

`policy-rag monitor` first checks that each signal predicts a retrieval miss or an out-of-scope question on the
labeled set; a drift alarm on a signal that doesn't predict failure is noise. It then runs a control: half the
reference against the other half, to see the false-alarm rate at this sample size. Finally it runs Evidently's
drift preset between the evaluated questions and a sample of 30 everyday queries, including topics such as prior
authorization and hospice that these chapters don't cover.

Queries that look low-confidence on two or more signals go to a review queue. Labeling them and adding them to the
test set is the feedback loop. It stops short of automatic re-tuning on purpose: without labels on live queries, an
automatic change could only optimize a proxy.

### What the sweep found

![Recall at 5 by chunk size, overlap and retriever](../../assets/policy-rag/sweep.png)

Across 15 settings, top-5 recall stayed between 88% and 94%, three questions apart at most. Two patterns are worth
following up:

- **Bigger chunks help everyday wording.** Embedding search goes from 70% to 80% on plain-language questions at
  800 words.
- **Smaller chunks rank better.** At 150 words, hybrid puts the right section first 82% of the time (76% now),
  on half the context.

The best setting within the context budget gained 2 points, with an interval of -6 to +12, so the rule kept the
current configuration. Both patterns go on the list for the next sweep, once there are more plain-language
questions to judge them on.

### What monitoring found

![AUC of each signal for retrieval misses and out-of-scope questions](../../assets/policy-rag/monitoring_signals.png)

Every confidence signal cleared the bar. The best embedding match was the strongest predictor of both a retrieval
miss (AUC 0.89) and an out-of-scope question (0.97). Keyword-based signals caught out-of-scope questions well but
misses poorly.

The control earned its keep on the first run. Two random halves of the same questions "drifted" on one signal at
p = 2.5e-8. That signal takes only the values 1-5, so Evidently had tested it with chi-square, and a value missing
from one half produces an expected count of zero and a p-value near zero. It is now tested like the other
numeric signals, and the control comes back clean. Without the control, that artifact would have counted toward a
drift alarm.

![Signals for the evaluated questions and the traffic sample](../../assets/policy-rag/monitoring.png)

The 30 traffic queries were shorter than the eval questions, rarely contained a code, and keyword and embedding
search agreed on them less often. Two of eight signals drifted, below the half-of-signals rule, so there was no
alarm. That is the right call at this sample size, but not a reassuring one: five of six confidence signals moved
toward lower confidence. Thirty queries is too few for the test to confirm it.

The review queue flagged 2 of 30 queries, both out of scope (hospice election, Part D penalties), and no in-scope
ones. Out-of-scope questions that share therapy vocabulary, such as Medicare Advantage prior authorization for PT,
looked confident to retrieval and weren't flagged. Those depend on the answer step declining, which it did for all
12 out-of-scope questions in the eval.

**What this changes.** At this volume, the review queue is more useful than the drift alarm. Next:

- normalize the keyword score by query length (part of its drift was just shorter queries);
- test drift on windows of a few hundred real queries, where the tests have power;
- track the decline rate on topic-adjacent questions.

## Engineering

- **One command per step:** `index`, `ask`, `eval --no-judge`, `eval --judge-only`, `calibrate`. Answers are
  generated first and graded afterwards, so the large judge model loads once instead of swapping per question.
- **Everything is logged to MLflow:** settings, lineage hashes, metrics, traces and reports, in a local SQLite
  store. There is no tracking server or model registry: it is one person on one laptop, and nothing here is trained.
- **A `--fake` mode** (hash embeddings and canned replies) runs the full pipeline and tests without a model server.
- **Grades are reused when answers repeat:** `scripts/reuse_judge.py` copies judge grades to answers identical to a
  graded run and judges only the rest. On the rerun that saved 27 of 62 answer grades and 34 of 62 closed-book ones.

## Limits

- **Four chapters, not the full manuals.** A production version would index all relevant chapters, local coverage
  determinations and articles, and refresh them when CMS revises a manual.
- **62 questions is small,** and only 10 are in plain language. Differences between search methods have wide
  intervals: one question is two points, and the rerank answer test's interval spanned −9 to +9.
- **I drafted the questions with an AI assistant** and checked each against its source. A reviewer-written test set
  would be a stronger benchmark.
- **The answers quote policy;** they are not billing or legal advice.

The code, questions, labels and reports are at [github.com/RonCom/policy-rag](https://github.com/RonCom/policy-rag).
