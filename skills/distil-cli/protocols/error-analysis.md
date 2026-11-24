# Protocol: Error Analysis

Turns a model's predictions into **findings**: named failure modes, counted, each with the step
that can fix it. Phase 4 runs it after every trained model to scope the next iteration.

A score alone does not say what to change. The useful findings are usually shapes of input
(“the message vents about one issue, then asks for another one first”, “a short message that uses
the words of a different label”), not label pairs, so read the errors, not only the counts.

## Inputs

- the predictions of the model under analysis on the agreed test set (`operations/train-model.md`
  § Pull the results, or an endpoint run from `operations/evaluate-deployment.md`);
- the teacher's predictions on the same test set (from the Phase 3 teacher evaluations; download
  them once into `teacher-evaluation/<name>/output/` and reuse them);
- the best model's predictions, from the run log's best ledger row;
- the row groups of the test set (named sets of rows, for example the user's rows, each generated
  batch) and the flagged-rows file, from the run log header.

Rows are matched across files on the last user message. With `base.variable_system_prompt` on,
see `references/variable-system-prompts.md`.

Write the code for these steps yourself, for this task's format and metric: a script that joins
the prediction files on the row input and computes the numbers below. Keep it in the run
directory.

## Step 1: Scores by group

Score the predictions with the project's metric (Phase 2): overall, for each row group, and
without the flagged rows. Use the per-row scores the platform's predictions file carries for its
metrics (`binary`, `rouge`, `llm-as-a-judge`, `llm-as-a-judge-reference-free`), or the user's scorer
for a metric the platform does not compute. A metric computed over the whole set (an F1 over
spans or classes) has to be computed per group, not averaged over rows.

A group that scores near the ceiling for every model, the teacher included, hides differences
between models; say so. The user's own rows usually track production better than generated rows,
so read them separately.

## Step 2: Compare with the teacher and the best model

Call a row correct when its score passes the metric's bar (an exact match, a judge verdict of
good, or a threshold you state for a graded score). Then count:

- **student against teacher:** rows both get right, only the student, only the teacher, neither.
  Rows only the teacher gets right can be fixed with training data. Rows neither gets right are
  beyond the teacher, and more data from it will not fix them;
- **against the best model:** rows fixed and rows broken. Many broken rows mean the last change
  shifted the training data away from something the model used to handle;
- **for labels:** the most common (reference → prediction) pairs, overall and per group. For free
  text there are no such pairs: read the rows the metric marked bad instead.

## Step 3: Read the errors and name the failure modes

Read every error on the user's rows, and a sample of the errors on generated rows, starting with
the rows only the teacher gets right. For each, write down the input's shape, the reference and
the prediction. Group them into named failure modes and count each. A mode with one example says
little; one with many is a pattern.

When a mode may come from the training data, search the Dataset the model trained on for it
(`references/execution/cli.md` § Reading a Dataset): the train rows may lack that shape, or teach
it wrong.

**Flag missing capabilities.** Some errors show a kind of input or a capability that the scope
(Phase 1) never listed and the test set barely contains. They show up in rows from outside the
test set (an evaluation on other data, production traffic), or as a small group of test rows that
fails as a whole. Count the rows of that kind in the test set and in the train set. When the test
set has too few to measure it, the problem is not the model but what the project measures: flag it
as a scope gap.

## Step 4: Map each mode to its cause and fix

| What the predictions show | Likely cause | Fix (Phase 4 change) |
|---|---|---|
| teacher right; the student is wrong on a shape the train rows barely cover | coverage | add rows of that shape: a new mutator value |
| teacher right; the student is wrong the same way everywhere | the train rows teach it wrong | fix the value or instruction that produces those rows, then add correct rows |
| teacher right; a targeted slice improved but other rows broke | the last top-up shifted the mix | add rows restoring the shrunk forms at their usual weights |
| teacher right; no pattern, small gap to the teacher | capacity or too little training | a training change on the same Dataset (another student, epochs) |
| teacher wrong where the student is wrong | the teacher cannot do it | no data fix; record it, it bounds the target |
| a kind of input the scope never listed, with few or no test rows | scope gap | add it to the scope and propose new test rows to the user (Phase 3 § Changing the test set later); then train rows for it |
| the reference looks wrong, or teacher and student agree against it | wrong test label | check the flagged-rows file; report it at the next stop |
| the judge marks correct answers bad, or the reverse | the measure is wrong | report it at the next stop; once agreed, the judge is not changed |

## Output: findings

Write the findings into the run log entry of the iteration: each failure mode with its count, an
example input, the cause, and the proposed fix, sorted by how many errors a single change could
remove. List the scope gaps separately: they need the user before the test set changes. Also record the targeted-slice score: the slice the last iteration aimed at, before and
after.
