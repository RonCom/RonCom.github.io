---
date: 2026-10-05
slug: minijev
authors:
  - chris
categories:
  - Generative AI
  - Evaluation
  - Calibration
description: Rebuilding a non-generative "System One" decision model on an 8 GB laptop GPU, trained with a proper scoring rule on synthetic data, and what moved its accuracy and calibration.
---

# minijev: a calibrated decision model on an 8 GB laptop GPU

TypeSafe AI's Jev is a language model that never writes text. You send it a `state` (a message, a document, an agent
transcript) and a set of named questions, and it returns a probability for every answer in one call. TypeSafe doesn't
publish its base model, training data or reward, so I rebuilt the idea from the outside: a small open model (Qwen3) on
an RTX 4060 Laptop GPU with 8 GB of VRAM, trained to give probabilities that mean what they say.

<!-- more -->

!!! abstract "TL;DR"
    - **Interface:** Jev's three question types (yes/no, multiple choice, rubric score), served on a local endpoint shaped like Jev's API.
    - **Method:** answers read from one forward pass over single-token answer codes; LoRA training with a proper scoring rule (log score + Brier) computed exactly over the answer distribution, with no sampling.
    - **Data:** synthetic rows written and checked by a local `qwen3:8b` teacher in Ollama, later mixed with 2,000 public training examples.
    - **Calibration:** 1,001 synthetic rows cut calibration error (ECE) from 0.247 to 0.032 on the 1.7B model.
    - **Best model so far:** Qwen3-4B in 4-bit, 81.8% accuracy and ECE 0.024 on 1,200 held-out questions, up from 72.3% / 0.247 for the untrained 1.7B.
    - **Ceiling:** an 8B teacher can't write hard NLI data. Doubling synthetic data moved ANLI from 55% to 57%; 1,000 real ANLI training examples moved it to 67%. Jev scores 81.4%.
    - **Status:** in progress. The code is in a private repository for now.

## What Jev does

Each request is a `state` plus named questions of three types:

- **Noul**: a yes/no question, answered with P(yes)
- **Choice**: one of up to 255 options, answered with a probability for each
- **Score**: a rubric of 2 to 10 levels, answered with a probability for each level

All questions are answered independently in one call. TypeSafe calls its training method Reinforcement Learning for
Calibrated Decisions. Their benchmarking paper ([arXiv:2609.37647](https://arxiv.org/abs/2609.37647)) reports
jev-1.13.0 at 96.4% on SST-2, 91.3% on BoolQ, 81.4% on ANLI-R1 and 88.5% on AG News. Those are the targets here.

## Design choice 1: no generation

Each question becomes a chat prompt that lists single-token answer codes (`Yes`/`No`, `A`, `B`, `C`, ..., `0`-`9`).
The model reads the prompt once, and the answer is the softmax of the next-token logits over those codes. Nothing is
sampled. This is the same method the benchmarking paper used for its open-model baselines, so my numbers sit on the
same scale as Jev's.

The untrained model already produces Jev-shaped output this way. Training is for accuracy and calibration.

## Design choice 2: reward honest probabilities, exactly

The loss is a proper scoring rule: log score plus Brier score, plus ranked probability score for rubric questions. A
proper scoring rule is maximized only when the stated probability equals the real chance of being right, so it pays
for calibration directly.

Each answer is a distribution over a handful of codes, so the expected reward can be computed exactly instead of
estimated from samples. That makes it the exact policy gradient. I chose it over GRPO: a right/wrong reward on a
sampled answer pushes all probability onto the top guess, which is the opposite of calibration.

## Design choice 3: synthetic data, label first

A local teacher (`qwen3:8b` in Ollama, on the same GPU) writes the training data across eleven task families:
sentiment, topic, intent routing, grounding, NLI, passage QA, moderation, prompt injection, agent monitoring, rubric
scoring and relation choice.

- **Label first.** The generator picks the correct answer, then asks the teacher to write a state that makes it right.
  That balances classes, and the label is what the text was written to satisfy.
- **Blind check.** A second call asks the teacher for probabilities without seeing the label. Where the two
  disagree, the row is kept with a split label instead of a hard one.
- **Fit the GPU.** The teacher and the trainer can't share 8 GB, so data is generated first, Ollama is stopped,
  then training runs. Generation runs at about 1 row a minute.

## Results

1,200 held-out questions, 300 each from SST-2, BoolQ, ANLI-R1 and AG News. ECE is expected calibration error: the gap
between stated confidence and actual accuracy, where 0 is perfect.

| Model | SST-2 | BoolQ | ANLI-R1 | AG News | Accuracy | ECE |
|---|---|---|---|---|---|---|
| Jev 1.13 (published) | 96.4% | 91.3% | 81.4% | 88.5% | | |
| Qwen3-1.7B, untrained | 91.7% | 76.0% | 46.3% | 75.3% | 72.3% | 0.247 |
| Qwen3-1.7B, 1,001 synthetic rows | 92.0% | 81.7% | 50.0% | 88.0% | 77.9% | 0.032 |
| Qwen3-4B, 1,001 synthetic rows | 90.7% | 85.3% | 55.0% | 84.7% | 78.9% | 0.036 |
| Qwen3-4B, 2,001 synthetic rows | 88.3% | 84.7% | 57.3% | 87.0% | 79.3% | 0.057 |
| Qwen3-4B, synthetic + 2,000 public rows[^public] | 90.3% | 85.7% | 67.0% | 84.3% | 81.8% | 0.024 |

[^public]: 1,000 examples each from the ANLI R1 and BoolQ train splits, which don't overlap the evaluation questions. This model isn't zero-shot on those two tasks.

With 300 questions per set, one set's accuracy has a standard error of about 2.8 points, so differences under 3
points between rows are noise. The 4B runs use 4-bit base weights (QLoRA) to fit in 8 GB.

## What I learned

- **Calibration came fast.** One epoch on 1,001 synthetic rows cut the 1.7B model's ECE from 0.247 to 0.032. When it
  says 80%, it's right about 80% of the time.
- **A second epoch overfit.** The 1.7B scored 77.8% / ECE 0.026 at the end of epoch 1 and 75.7% / 0.078 at the end of
  epoch 2. Training now keeps the checkpoint with the lowest held-out loss.
- **The 4B helped on reasoning.** On the same data it gained 5 points on ANLI and 3.6 on BoolQ, and its yes/no ranking
  (AUROC) rose from 0.87 to 0.92.
- **The teacher set the ceiling.** Doubling the synthetic data, with over half the new rows aimed at NLI, passage QA
  and grounding, moved ANLI only from 55% to 57%. An 8B teacher can't reliably write or check adversarial NLI: 32% of
  all rows ended up with split labels because its two passes disagreed.
- **Real labels broke through.** 1,000 ANLI training examples moved ANLI from 57% to 67% in one run.
- **Topic classification matched Jev** from synthetic data alone: 88.0% on AG News against Jev's 88.5%.

## Limits

- **Four benchmarks, not 37.** The paper evaluates Jev on 37 datasets. Mine covers four, with 300 questions each.
- **The best checkpoint is picked on the evaluation set,** from two to four candidates per run, which flatters the
  numbers slightly.
- **One teacher.** Every synthetic row comes from the same 8B model that also checks it.

## Next steps

1. **More public ANLI data.** The ANLI R1 train split has about 17,000 examples, and the best run used 1,000.
2. **A stronger teacher** for the NLI and passage-QA rows, to keep the training data purely synthetic and test
   whether teacher quality is the limit.
3. **Encode the state once.** Each question is a separate forward pass today; reusing the shared state's KV cache
   would make many-question requests cheaper.
4. **More of Jev's benchmark,** starting with MMLU.
