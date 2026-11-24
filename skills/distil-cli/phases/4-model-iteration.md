# Phase 4: Model Iteration

| | |
|---|---|
| **Input** | the best Dataset so far (the test-set Dataset in iteration 1); the best model; the findings of the last error analysis |
| **Output** | one ledger row; the best Dataset and best model updated; new findings |
| **Gate** | none: the user is not involved in the loop |
| **Uses** | `protocols/make-rows.md`, `operations/train-model.md`, `protocols/error-analysis.md`, `protocols/run-log.md` |

This file is one iteration. `SKILL.md` § Phase 4 runs it again and again and owns the stop
conditions. One iteration adds rows (or, rarely, changes training), trains, and finds what to fix
next.

## Step 1: Scope the rows to add

**Iteration 1, the initial train set.** With traces in the Dataset, the first rows come from the
traces (`source=traces`): real inputs set the format and the hard cases that generation then
imitates. Without traces, generate the initial train set (`source=synthgen`) with the size agreed in
Phase 2. Write its generation settings for the training data: mutators spread over the whole
input inventory and every output, so the rows cover the scope. You may reuse lessons from the
test-set smokes (instructions that fixed label errors, validator checks), but not the test set's
per-gap mutator values themselves: when train and test rows come from the same settings, the test
rows stop measuring anything. With train rows the user uploaded, they are already in the split;
add to them.

**Later iterations.** Take the largest finding from the last error analysis that rows can fix,
and write the hypothesis in one sentence (“the train rows have no inputs that vent about one issue
and then ask for another one first”). Choose **one** change, usually a new mutator value for the
missing shape, or a corrected value or instruction that taught something wrong. A top-up teaches
whatever its rows have in common, not only the rule you meant, so design it so the student cannot
learn a shortcut: next to the rows for the target shape, include look-alikes that share its
surface features (wording, framing, topic) but get the other answer, so only the deciding fact
separates them. Step 4 then checks both sides: the targeted group, and the rows near it that
should keep their answer.

**After a labelling rule changed** (the run log records the change and the new test-set
version), the train rows made under the old rule still teach it. Before adding rows for anything
else, correct the rows the rule applies to as a local fix (`SKILL.md` § Rules), or drop them and
make new ones under the new rule.

Where new rows come from: synthetic generation, and relabelling while traces are left. Never
downloaded datasets or rows you write (`SKILL.md` § Rules).

**A training change instead** (another student, more epochs) only when the error analysis points
at capacity: the teacher is right, the errors have no pattern, and the gap to the teacher is small.
It trains on the best Dataset; Steps 2 and 3 become one training run.

## Step 2: Make the rows

Run `protocols/make-rows.md` with split `train`, the source from Step 1, the **best Dataset** as
parent, and the size from Phase 2: the full first-train-set size in iteration 1, the top-up size
(500–1,000 rows aimed at the failure mode) from iteration 2 on. The new Dataset holds the old train
rows and the new ones, and the same test set.

## Step 3: Train

Train on the new Dataset (`operations/train-model.md`) with the best model's training settings
(defaults in iteration 1). Price it first: the price grows with the train set.

**A range of students** (Phase 2): in iteration 1, train each student. From iteration 2, continue
with the best one by score; when two are within the noise, the smaller one.

**Training noise.** Comparing two trained models needs the training noise, which the teacher
evaluations do not measure (`references/evaluation-metrics.md` § Verdicts). When the budget
allows, train the first model twice with different `random_seed` values in iteration 1. Otherwise,
retrain once with another seed the first time a decision is close.

## Step 4: Compare and decide

Score the model on the primary metric, overall and per row group (`protocols/error-analysis.md`
Step 1), and fill its ledger row. Then decide whether to keep the new Dataset. Weigh three things:

- the group the change aimed at: did it clearly improve? And the rows like it that should keep
  their answer: did they hold?
- the real-row groups (the user's rows, relabelled traces), which show what production will see:
  did anything get worse by more than the noise?
- the overall score.

Use the noise that fits the comparison: the training noise when you compare two trained models,
and a wider margin for small or unstable groups (`references/evaluation-metrics.md` § Verdicts).
These are guidelines, not a formula, so use judgement and write down your reasoning:

- **Keep** the new Dataset when the targeted group clearly improved and nothing that matters got
  worse beyond the noise; more correct data helps the student generalise. The model becomes the
  best model when it is better overall, or as good overall and better on the groups that matter.
- **Roll back** when the change made the model noticeably worse where it matters. The best
  Dataset and best model stay as they were, and the next iteration forks from the best Dataset.
  Record the rolled-back Dataset in the lineage.
- **A change that moved nothing**, neither the targeted group nor anything else beyond the
  noise: keep the Dataset (the rows are correct, and more correct data helps), keep the best
  model as it is, and record that the hypothesis did not hold.
- **When it is close**, for example the targeted group improved but a real-row group lost a few
  rows, look at which rows moved. The same borderline rows flipping between models is training
  noise. Retrain once with another seed before you roll back a change that fixed what it aimed
  at.

## Step 5: Error analysis

Run `protocols/error-analysis.md` on the new model's predictions. Check the targeted slice first:
did the rows the hypothesis was about improve? Then the rows fixed and broken against the best
model, and the failure modes left. The findings scope the next iteration.

## Record

The ledger row, the lineage rows, the findings, the spending. `SKILL.md` § Phase 4 then checks
the stop conditions.
