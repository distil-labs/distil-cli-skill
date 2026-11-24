# Phase 3: Test Set

| | |
|---|---|
| **Input** | the scope and its input inventory (Phase 1); the agreement, the agreed job description and the root Dataset (Phase 2) |
| **Output** | the agreed test-set Dataset; its row groups and flagged rows; the target, the noise, the production baseline |
| **Gate** | the user approves any expansion, and agrees to the test set |
| **Uses** | `protocols/make-rows.md`, `operations/teacher-evaluation.md` |

## Goal

A test set that covers the scope found in Phases 1 and 2, with labels you can trust. Every
verdict in the project is read on it: whether an iteration helped, when the loop stops, how the
model compares with production. A kind of input the test set lacks cannot be measured, so the loop
cannot see it fail.

The test set starts from real rows: the user's labelled rows, and relabelled traces. Generated
rows are added only for the gaps that the coverage check below finds. Rows generated for no
particular gap, or with the same settings that will later generate the training rows, measure
little: the student learns that generator's style and scores near 1.0 on such rows, whatever its
real quality.

## Step 1: Take stock

Download the root Dataset and list the test rows you have: the user's labelled rows, and the
traces that could become test rows. Without traces, split the user's labelled rows about evenly
between test and train, so that generation has real rows to follow; agree the split with the
user. When there are traces, run `protocols/make-rows.md` with
source `traces` to turn them into test rows.

## Step 2: Check the coverage against the scope

Compare the test rows with the scope, item by item, and count:

- **each input kind in the Phase 1 inventory** (source, format, language, structure such as list
  length): how many test rows show it;
- **each output** (label, field or call type): how many test rows have it as the answer;
- **each rule, threshold and edge case in the job description**: how many test rows depend on it,
  including rows on both sides of each threshold;
- **out-of-scope inputs**: how many rows, and of which kinds.

A gap is an item with no rows, or with too few rows to show a difference between two models (fewer
than about 15–20). Write the coverage table into the run log, gaps included.

## Step 3: Propose the expansion to the user

Present the coverage table and the gaps, and for each gap how to fill it and roughly how many rows:

- **relabelled traces** of that kind, when there are any (real inputs, the closest to production);
- **generated rows** for that kind, otherwise.

Include the price. The test set as a whole should end with a few hundred rows (between 100 and
1,000): more rows narrow the noise, but past a thousand the gain is small. If the user considers
their own rows the gold standard and does not want generated rows, keep the test set as it is and
record the gaps as known blind spots.

◆ The user approves the expansion, or declines it.

## Step 4: Fill the gaps

Run `protocols/make-rows.md` on the test split. Each gap gets its own generation settings: mutator
values that describe that input kind, written for this purpose. Do not reuse a single general set of
settings across the gaps, and later do not generate training rows with the same settings (Phase 4
writes its own). The full run's sample review must pass before the test set goes to the user. If the
reviews keep failing after the settings are settled, the teacher is the limit: propose a stronger
test-row teacher to the user.

Record the row groups: the user's rows, the relabelled rows, and one group per filled gap.
Scores are reported per group, because the user's rows usually track production best, and a group
that scores near 1.0 for every model measures nothing. List the rows the review flagged as
undecidable or debatable in a flagged-rows file.

## Step 5: Agreement, target and noise

◆ The user agrees to the test set. Once agreed, it is not changed, except as § Changing the
test set later describes. Then, on the agreed test-set Dataset:

- run two teacher evaluations with the agreed teacher (`operations/teacher-evaluation.md`). Their
  spread, overall and per row group, is the **evaluation noise**. Note any group whose two runs
  differ much more than the others: it is unstable, and its scores need a wider margin
  (`references/evaluation-metrics.md` § Verdicts). The training noise is measured in Phase 4;
- set the bar with other models. The agreed teacher wrote the references of the relabelled and
  generated rows, so its score on them is inflated. Run one evaluation with each of one to three
  other strong models (proprietary models are allowed for teacher evaluation,
  `references/model-catalog.md` § Teacher models). The **target** is their mean;
- run one evaluation with the production model, if there is one. That is the **baseline**, a floor
  the student must clear before it is deployed.

Present the target, the evaluation noise (and any unstable group) and the baseline to the
user.

## Changing the test set later

New gaps can show up later: in an error analysis (Phase 4), in the final evaluation (Phase 5),
or when the user describes inputs that were not in the inventory. Add the new input kinds to the
scope, run Steps 2–4 again for them, and ask the user before you change the test set. A new test
set makes every earlier score incomparable, so it is the user's decision.

When the user agrees, build the new version on a Dataset that carries the current train rows
(upload the best Dataset's files with the new test rows, or expand it), and re-run Step 5. Measure
the best model on the new set through an endpoint, without retraining
(`operations/evaluate-deployment.md`), and start a new ledger section.

**A changed labelling rule.** When the user adds or changes a rule, write it into the job
description first (`task_description`, and the judge rubric when it is affected), as a new version
recorded in the run log. The test rows the rule applies to may now have wrong references. Find them
and write each new reference under the new rule, then check each change with a second, independent
pass (`protocols/blind-review.md`): apply a change when both passes agree, and list the rows they
disagree on for the user. The user's own rows change only when the user agrees. Apply the changes as
a local fix (`SKILL.md` § Rules): download the test set, correct the rows, and upload a new Dataset
with the train rows. Run Step 2 again for the new rule, with rows on both sides of it, and agree the
new version with the user as above. Train rows made under the old rule still teach it; Phase 4
corrects them first.

## Record

The test-set Dataset id, the coverage table, the row groups, the flagged-rows file, the target,
the noise and the baseline. Mark Phase 3 done.
