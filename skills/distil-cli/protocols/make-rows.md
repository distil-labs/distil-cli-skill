# Protocol: Make Rows

Adds verified rows to one split of a Dataset. It is the one routine both data phases use:
Phase 3 for test rows, Phase 4 for train rows. The rows come from one of two sources, `traces`
(the Dataset's real inputs, labelled by the teacher) or `synthgen` (new rows the teacher
generates), run through the same loop: a small smoke run, a blind review of it, a fix of whatever caused the
errors, and again, until a smoke passes; then the full run.

```
 settings ─► smoke (100–250 rows) ─► blind review ─┬─ pass ─► full run ─► (test split: sample review) ─► new Dataset
     ▲                                            └─ fail ─► fix the cause ─┐
     └──────────────────────────────────────────────────────────────────────┘
```

The point of the loop is that a full run is expensive, and problems are cheapest to find on a few
hundred rows. Fixing the settings that produced them fixes every later run too. What the settings
cannot fix can still be filtered or corrected locally after the full run (Step 6).

## Inputs

| Input | Values |
|---|---|
| split | `test` or `train` |
| source | `traces`: real inputs from the Dataset's traces, with answers written by the teacher (the relabelling job); or `synthgen`: new rows the teacher generates (the synthetic data generation job) |
| parent | the Dataset to add rows to: the root Dataset or the test-set Dataset (Phase 3), the best Dataset so far (Phase 4) |
| target | rows to add in the full run: `trace_processing.num_<split>_relabelled` or `synthgen.<split>_generation_target` |
| what the rows are for | the slices to cover, in one or two sentences; the reason they are needed |

Thresholds and run sizes come from the run log header.

## Step 1: Write the settings

Read `operations/relabel-traces.md` (source `traces`) or `operations/generate-synthetic-data.md`
(source `synthgen`) for what each setting does and how to send it, and `references/mutators.md`
for the `synthgen` patterns below. The settings are a config override and the job description on
the parent; keep them in the run directory's `input/`.

### Source `traces`: the settings that decide quality

- **The relabelling teacher** (`trace_processing.teacher_model_name`, or a committee with
  `trace_processing.relabelling_committee_models`) writes the answers. On the test split these are
  the reference answers every score is read against, so use the strongest teacher the agreement
  allows (the test-row teacher, if there is one). A committee keeps answers that several models
  agree on. Set a new teacher in both teacher fields (see the operation file).
- **Relabelling instructions** (`trace_processing_instructions`): what the teacher must know about
  these traces to answer them by the job description. Examples are their format, truncated inputs,
  and habits of the production model to ignore. They are not new rules: rules belong in the task
  description.
- **Relevance filtering** (`trace_processing.relevance_filtering` and its thresholds) drops traces
  that do not belong to this task at all. Use it when the traces of several applications or
  agents are mixed together, so that only this task's traces become rows. It does not decide
  what is in or out of scope: inputs the user named as out of scope still belong to the task, and
  the model must learn to answer them. It is off by default.

### Source `synthgen`: the settings that decide quality

- **Mutators** set what each row is about. Use one value per label or per slice you need, and
  give each dimension (label, message form, language, …) its own mutator with weights. A case
  the teacher gets wrong when it must work the answer out can be its own value that fixes the
  deciding facts and the answer together, as long as the value fixes everything the answer
  depends on (`references/mutators.md` § Patterns that work).
- **Generation instructions** (`synthetic_data_generation_instructions`): every deciding fact is
  stated once and consistently; inputs never state the conclusion (“it is before the cutoff”)
  or name the answer; numbers come from safe ranges away from thresholds unless a mutator value
  asks for a near-threshold case; rows in one batch differ in structure, not only in numbers.
  When the split has no real rows to follow, describe the input and output format in the
  instructions, with an example.
- **LLM validator** (`synthgen.validation_instructions`): a second pass that re-derives the
  answer, computes the deciding facts, and rejects rows whose answer differs, whose facts are
  missing or contradict each other, or that state the conclusion. It catches part of the
  teacher's errors at the cost of some rejected rows.
- **When a slice is being added on top of existing rows** (Phase 4), set the weights for the
  mix of the whole train set after the top-up, and add look-alikes with the other answer
  (`references/mutators.md` § Patterns that work).

## Step 2: Smoke

Price it with `--dry-run`, add the price to the run log, and submit the smoke on the **same
parent** as the full run, at the smoke size (`references/platform.md` § Smoke runs). A smoke writes
a side branch, so the parent keeps everything it holds. Never continue from a smoke's Dataset: the
full run goes on the parent. Download the smoke's Dataset into the run directory's `output/`.

- **Source `traces`:** set `trace_processing.num_<split>_relabelled` to the smoke size. The job
  refuses to run when the parent holds fewer distinct traces than that. The price is the relabelled
  traces times their average tokens, once per teacher in a committee.
- **Source `synthgen`:** set `synthgen.<split>_generation_target` to the smoke size. For a smoke
  that tests a new mutator value, put most of the weight (about 70%) on that value, so the review
  sees enough of its rows. The price is the target times the average tokens of a row in the parent,
  and a Dataset made by an expand carries the task description inside every row. When the parent's
  rows are long, a smoke on an earlier Dataset with the same example rows (for example the uploaded
  root) costs a fraction and tests the same settings. Use that for the first train set. For a
  top-up, smoke on the real parent, because the rows it already holds shape what it generates.

## Step 3: Review the smoke

Run `protocols/blind-review.md` on all the rows the smoke added. Then check what the review does
not cover:

- **Source `traces`:**
  - the share of traces the job dropped (the job log says at which step);
  - the mix of inputs against the Phase 1 inventory, so you know which kinds the traces cover;
  - the rows where the teacher's answer differs from the production answer in the trace. Read
    them first: either the teacher or the production model is wrong there.

  The inputs are real, so a reviewer's "unrealistic" rating is usually noise in the trace itself
  (a truncated input, a test message), not a setting to fix.
- **Source `synthgen`:**
  - the share of rows per requested value, since a mutator value can produce nothing;
  - near-duplicate inputs;
  - the length and language mix against what the slice needs.

## Step 4: Fix and repeat

If the review fails, change the setting that caused each error pattern and run a new smoke
(`smoke-2`, `smoke-3`, … on the same parent). Change one cause per smoke when you can, so the next
review shows what the change did.

**Source `traces`:**

| What the review shows | Fix |
|---|---|
| the answers copy habits of the production model | relabelling instructions: say what to ignore; or a stronger teacher, or a committee |
| the answers are wrong on one kind of trace (long documents, a language, a format) | relabelling instructions about that kind; a stronger teacher |
| far fewer rows than traces | the job log says which step dropped them; with relevance filtering on, check that its thresholds are not dropping this task's traces |
| the rows do not contain the kind of input you need | source `synthgen` for that kind |
| the error rate stays above the threshold after the settings are settled | a stronger teacher or a committee for this split (a user decision; `SKILL.md` Phase 2) |

**Source `synthgen`:**

| What the review shows | Fix |
|---|---|
| a mutator value produces contradictory or mislabelled rows | reword that value; name the facts it must state |
| rows lack a fact that decides the answer | instructions: always state it (an exact time, a day, an amount) |
| the teacher miscounts near a threshold | safe ranges in the instructions, plus explicit near-threshold values with the outcome |
| rows state the conclusion or name the answer | instructions and a validator check against it |
| one answer wins over another against the task's priority rule | a value per winning case; spell the rule out in the instructions |
| the error rate stays above the threshold after the settings are settled | a stronger teacher for this split (a user decision; `SKILL.md` Phase 2) |

**Fast path**: settings that already passed a smoke in this project go straight to the full run.

**When time or budget runs short.** After a failed smoke, it is tempting to send the fixed
settings straight to the full run. A fix that was never checked often fixes less than expected,
and sometimes breaks something else. A relaxed validator, for example, can stop checking what the
original one caught. Choose what fits the situation, and record the choice in the run log:

- run the full run, then blind-review a random sample of its rows before using them. A review
  costs time, not platform money, and it checks the fix on the rows you will actually use;
- make the run smaller, for example only the languages or forms the passing part of the smoke
  covered;
- drop the change for this iteration, and use the time on something already verified.

Whichever you choose, do not use rows whose fix was never checked. A local filter (Step 6)
removes the defects you know about. It does not show that the fix worked.

## Step 5: Full run

Price it with `--dry-run`, add the price to the run log, and submit it on the parent with the
planned count (`num_<split>_relabelled` for `traces`, `<split>_generation_target` for `synthgen`).
Download the result into `full-N/output/` and check the row count against the plan. Relabelling
drops traces that fail its filters or validation, and generation drops duplicates and rows the
validator rejects; the job log says how many (`references/execution/cli.md` § Monitor). For
`traces`, the full run spends those traces: they are gone from the Datasets after it.

- **Test split**: blind-review a random sample of the new rows (the sample size from the run log
  header). The test set decides every later verdict, so its labels need the check. If the sample
  fails, do not present the test set to the user: fix the settings and run again from the parent.
- **Train split**: no review of the full run when its settings are the ones that passed the smoke;
  training is the check. When the settings changed after the last passing smoke, review a
  sample first (Step 4, "When time or budget runs short").

## Step 6 (optional): Filter or fix locally

When the full run's rows have a defect the settings could not remove, or a check is easier as code
than as a validator instruction (JSON that does not parse, an answer naming a tool that is not in
the row, a label the review proved wrong), download the Dataset, run a script that drops or fixes
those rows, and upload the result (`operations/create-dataset.md`). Keep the script in the run
directory, and write what it changed and the counts before and after into the run log. Later
expands continue from the uploaded Dataset.

## Output

The new Dataset id, recorded in the run's `run.md` and in the run log's lineage with its row
counts and review results. For a test split, list the rows that a review flagged as undecidable
or debatable in a flagged-rows file, so scores can be reported with and without them (or drop
them in Step 6).
