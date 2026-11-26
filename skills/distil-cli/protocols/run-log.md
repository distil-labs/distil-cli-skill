# Protocol: The Run Log

The run log is the project's state. Phases do not link to each other; they read what earlier
phases wrote here and write their own outputs back. It is also how work resumes after a context
reset, and how the user sees what was done and why. Read it before every decision, and write to
it as things happen, not at the end.

## The project directory

One directory per model, named after it. Every job gets its own run directory under the
operation or protocol that ran it.

```
<model-name>/
├── log.md                                  the run log (this file's format)
├── raw-inputs/                             the user's files as received (read only)
├── dataset-uploads/<name>/                 each directory uploaded as a Dataset
├── teacher-evaluation/<name>/              input/, output/, run.md
├── relabel-traces-<split>/<smoke-N|full-N>/        input/, output/, review/, run.md
├── synthetic-data-generation-<split>/<smoke-N|full-N>/   input/, output/, review/, run.md
├── model-training/<name>/                  input/, output/, run.md
└── evaluate-deployment/<name>/             preds/, scores/, run.md
```

A run directory's `run.md` holds what that run needed and produced: the parent Dataset id, the
change it tests, the price from the dry run, the new entity id, and its result (review counts,
scores). `input/` holds the exact config and job description sent, `output/` what was pulled
back, `review/` the blind-review batches, labels and summary.

## log.md

Five sections, in this order.

### 1. Header

The agreement and the current reference points. Phases update it; it is the first thing every
phase reads.

```markdown
# <model name>

Entry: A (endpoint <unique-name>) | B (upload)   ·   Backend: CLI | API
Task type: …   ·   Teacher: …   ·   Test-row teacher: … (if different)   ·   Student(s): …
Metric: llm-as-a-judge-reference-free   ·   Judge: <model>, rubric in job_description.json
Budget: $… and … iterations   ·   Spent: $…
Review thresholds: label errors ≤ …%, undecidable ≤ …%, test-set sample … rows
Run sizes: smoke … rows, first train set … rows, top-up … rows
Scope: <scope file> (input inventory, rules, open questions) · Coverage table: <file>

Phase status: 0 done · 1 done · 2 done (job description v3) · 3 done · 4 iteration 3 · 5 -
Root Dataset: <id> (agreed job description)
Test set: <Dataset id> (<n> rows; groups: <names and counts>) · flagged rows: <file>
Target: <mean of the bar-setting models> (<model>: <score>, …) · Teacher score: <run 1>, <run 2> · Noise: evaluation ±<spread>, training ±<spread or "not measured">; unstable groups: <names> · Base student: <score>
Production model: <score or "none">
Best model: <SLM id> (<score>) · Best Dataset: <id>
```

### 2. Rules

The standing rules from `SKILL.md` § Rules, plus any project-specific rule the user adds
(for example "the train data must come from the spec teacher"). Write each with its date.

### 3. Lineage

One row per platform entity, so any Dataset or model can be traced back to its inputs.

```markdown
| # | Entity | Id | Parent | What it is | Cost $ | Status |
|---|---|---|---|---|---|---|
| D0 | Dataset (upload) | … | none | raw-inputs converted, draft job description | 0 | done |
| TE1 | TeacherEvaluation | … | D0 | teacher on the user's rows, job description v1, run 1 | 2.21 | 0.91 |
```

Uploads have no parent on the platform: write what they were built from in "What it is".

### 4. Ledger

One row per trained model, one section per test-set version (scores on different test sets are
not comparable). The section header repeats the target, noise and base score it is read against.

```markdown
## Ledger, test set <id> (<n> rows)
Target … · Noise: evaluation ±…, training ±… · Base … · closed = (score − base) / (target − base)

| Iteration | Hypothesis | Change | Trained on | SLM | Score | By group | closed | vs best | Cost $ | Kept |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | initial train set | relabelled 200 traces + 5,000 generated | <Dataset id> (5,200 rows; review 99%) | … | 0.74 | real 0.64 · gen 0.95 | 0.42 | best | 36 | yes |
| 2 | … | … | … | … | 0.86 | … | 0.85 | +0.12 | 180 | yes |
```

- **Hypothesis**: the failure mode the change targets, from the error analysis.
- **Change**: the one setting changed, in a few words.
- **Trained on**: the Dataset id, its train row count, and the review result of the new rows.
- **By group**: the score on each row group of the test set (`protocols/error-analysis.md`).
- **vs best**: the score minus the best kept score. **Kept**: `yes` when the new Dataset is
  kept (better, or within the noise), `rolled back` when it is noticeably worse.
- Write the row when the training is submitted, with its price; fill the scores when it ends.
  A row without scores is a run in flight.

### 5. Log

Dated entries, oldest first: what was done, why, and the outcome with absolute numbers
("0.864 → 0.903", not "+0.039"). Include every user decision and its wording when it changes
the plan, and every feedback report sent, with its sentiment and a one-line summary, so a resumed
session does not send it again.

## Resuming

Read the header's phase status. The first phase that is not done is where the work
continues. Then check the lineage for jobs still running (`references/execution/cli.md`
§ Monitor) and the ledger for a row without scores: finish those before starting anything new.
