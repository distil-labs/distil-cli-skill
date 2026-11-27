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
real quality. With no real rows at all (Entry C), follow § No real rows.

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

## No real rows

When the user brought no rows (Entry C), every item of the scope is a gap, and Steps 2 to 4 fill
them all with generated rows. Two things keep such a test set a measure:

- **A different teacher from training.** Generate the test rows with a test-row teacher other
  than the agreed teacher, which later generates the training rows. Otherwise the student learns
  the one generator's style and scores near 1.0, as § Goal says. Record which teacher made them.
- **The user checks a sample.** Before the agreement in Step 5, show the user about 20 test rows,
  spread over the outputs and the input kinds. Ask which answers are wrong and which inputs
  production would never send. Correct the rows they flag (a local fix, `SKILL.md` § Rules), and
  when several share a cause, fix the settings that made them and generate those rows again. The
  checked rows are their own row group. Where later phases read the real-row groups, read this
  group.

**Start small and iterate.** Without real rows, the first settings are a guess at what the user
means, and each check by the user corrects that guess. Small runs make the checks cheap and quick:

- **The first round is a smoke run** (`protocols/make-rows.md`): a few rows per gap, enough for
  the user's sample. Expect the user to find errors in the first rounds: wrong answers, inputs
  that look nothing like theirs, cases the scope missed. Each finding is a change to the
  settings, or to the scope with the user's agreement, then another small round.
- **Grow the test set only when a round passes**: the user finds nothing to correct, or only
  single rows with no shared cause. Then generate the full size, and review a sample as usual.
- **Keep the first train set small too.** Iteration 1 trains on a few hundred rows (agreed in
  Phase 2), not thousands. A first model trained quickly shows its errors early, and the error
  analysis often finds scope gaps that the user's description left out. Grow the train set in
  the later iterations, once the scope holds.

Record in the run log that the test set is generated, as a known limitation, and say so with
every score you report. Real rows lift it: the serving endpoint's traffic (Phase 5) gives traces
for the next cycle.

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
