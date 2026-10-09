# Workflow: Model Iterations

The inner loop: improving a trained model whose test set is locked, by re-entering
`build-a-model.md` at Step 8, 7 or 6 with one change at a time, until the model reaches an
agreed target. Step 5 (the test set) is re-entered only when the test set cannot measure the
failure or its reference answers are wrong, and only with the user's approval. The last model's predictions say what to change;
each stage lists what can be changed in it, in its § What Can Be Changed to Improve the Next
Iteration.

## Workflow Map

Print this map to the user when starting the workflow, before the first step:

```
 0 Agree the target, the budget and the stop conditions with the user
   ▼
 1 Observe ─── scores of the latest model, against the teacher, the base student,
   │           the production model and the previous iteration, on one test set
   ▼  stop? ─── target reached / budget spent / plateau / needs the user ──► report
   ▼
 2 Inspect ─── download the student's and the teacher's predictions; group the errors
   ▼
 3 Diagnose ─── each failure mode → the earliest step that can fix it
   ▼
 4 Set up ─── one hypothesis; changed settings for that step; record it in the ledger
   ▼
 5 Run ─── re-enter build-a-model at
   │         8  training only, same Dataset           (cheapest)
   │         7  the train set: relabel / generate again, or a top-up
   │         6  another teacher, then 7 and 8
   │         5  the test set: user approval; every earlier score becomes stale
   └──────────► back to 1
```

## The Ledger

Keep `iterations.md` in the project root, one row per iteration, and read it before every
decision. It is what lets the loop continue after a context reset.

```markdown
| Iteration | Hypothesis | Change | Re-entered at | Trained on | SLM | Primary metric | closed | vs best | Kept |
|---|---|---|---|---|---|---|---|---|---|
| 1 | first model | none | n/a | dataset … | slm … | 0.71 | 0.52 | n/a | yes |
| 2 | refund questions under-covered | mutator `topic` with 3 refund values, top-up of 2000 | Step 7 | dataset … | slm … | 0.78 | 0.69 | +0.07 | yes |
```

The columns: **Re-entered at** is the `build-a-model.md` step the iteration re-ran from;
**Trained on** is the Dataset id passed to training; **Primary metric** is the tuned student's
score on the agreed key; **closed** is from `../references/evaluation-metrics.md` § Verdicts;
**vs best** is the primary metric minus the best kept iteration's; **Kept** is yes, no, or
"within noise" (kept as the new base when not worse, but not progress for the plateau rule).

The header records the test-set Dataset id, the row count of its `test.jsonl`, the chosen
teacher and the baseline (from `test-set.md`). Rows below it share those rows: a later Dataset
in the chain, or one created at the outer loop's transition with the same `test.jsonl`, keeps
the section. When the test rows change, start a new section with the new id, because the
scores above it are stale. Each stage run also writes a new iteration directory in its own
stage directory, as the stage files describe. The ledger links them.

## Step 0: Agree the Target, the Budget and the Stop Conditions

Before the first autonomous iteration, agree with the user and write into the ledger's header:

- **The primary metric**: the task's default (`../references/evaluation-metrics.md` § Primary
  metric per task) unless the user names another. Every score in the ledger is this key.
- **The target**: a deploy candidate, that is `closed` ≥ 0.8 and above the production model's
  score from `../stages/teacher-evaluation.md` when there is one
  (`../references/evaluation-metrics.md` § Verdicts), unless the user specifies another one,
  for example the teacher's score (`closed` ≥ 1). A target the user gives takes precedence.
- **The budget**: the credits per route the loop may spend (`../references/platform.md`
  § Credits), and a maximum number of iterations. Every iteration spends at least one
  `slms_from_datasets_post` credit, so propose the iteration count from the balances.
- **The noise level**: run-to-run variation is measured, not assumed. Re-run one evaluation on
  the same model and test set (for example a second teacher evaluation), and treat any
  difference smaller than the spread between the runs as noise
  (`../references/evaluation-metrics.md` § Verdicts).
- **The plateau rule**: stop when two iterations in a row change the primary metric by less
  than the measured noise.
- **The judge**: set `evaluation.llm_as_a_judge_model_name` and `llm_as_a_judge_instructions`
  once and keep them for every run, so every score in the ledger is read the same way.
  Classification is scored by accuracy and has no judge; skip this for it.

Within that agreement the agent iterates without asking. It must stop and ask before:

- changing `task_description` or `classes_description`
  (`../references/job-description.md` § What each field feeds);
- changing the test set (re-entering Step 5);
- changing the judge model or the judge instructions;
- spending beyond the budget, or on a route the budget does not cover;
- moving production traffic to a model.

Deploying a model only to try it on traffic is allowed within the budget.

## Step 1: Observe

Read the latest model's scores (`../stages/model-training.md` Step 7) on the task's primary
metric (`../references/evaluation-metrics.md` § Primary metric per task) and put them next to:

- the teacher's score on the same test set (the ceiling);
- the base student's score (the floor) and `closed`;
- the production model's score from its teacher evaluation (a floor the model must beat);
- the previous iteration's score, from the ledger.

Compare only scores measured on the same test set. A difference smaller than the noise
measured in Step 0 is "within noise" in the ledger: the change is kept when it is not worse,
but it counts for the plateau rule, not as progress.

Then check the stop conditions. Stop and report to the user when the target is reached, the
budget is spent, the plateau rule fires, or the diagnosis in Step 3 points at something only
the user may change.

## Step 2: Inspect the Predictions

Download the student's predictions and the teacher evaluation's predictions on the same test
set (`../references/execution/cli.md` § Fetch metrics). Match rows on the last user message, not on
the whole prompt: the two prompts differ in their few-shot examples, and one test row can expand
to several evaluation rows, one per assistant turn. For each row the student got wrong, record
what the student answered, what the reference says, and whether the teacher got it right.

Group the errors into named failure modes: specific scenarios ("refunds for annual plans",
"dates in European format", "calls `search` when `lookup` was needed"), not "low score". Count
each mode. A mode with one example is noise; a mode with many is a target.

## Step 3: Diagnose

Map each failure mode to the earliest step of `build-a-model.md` that can fix it:

| What the predictions show | Likely cause | Re-enter at |
|---|---|---|
| The reference answer itself looks wrong, or the teacher and the student are both wrong in the same way | wrong test labels | Step 5: the user corrects the rows (§ Changing the test set) |
| The teacher is wrong where the student is wrong | the teacher cannot do it | Step 6: another teacher (`../stages/teacher-evaluation.md`), then the train set |
| The judge marks correct answers bad, or wrong answers good | the measure is wrong | ask the user, then Step 6: a new judge model or new judge instructions change what every score means, so every teacher evaluation runs again and every earlier student score is stale, as with a new test set |
| The teacher is right; the student is wrong on a slice the training data barely covers | coverage | Step 7: a top-up with mutators for the slice (`../stages/build-a-train-set.md` § Topping Up) |
| The teacher is right; the student is wrong the same way everywhere (format, a misread rule) | the training data teaches it wrong | Step 7: generation instructions or a stronger teacher (`../stages/synthetic-data-generation.md`), or relabelling when the relabelled rows carry the mistake (`../stages/relabel-traces.md`) |
| The teacher is right; the errors are spread with no pattern, and the gap to the teacher is small | capacity or too little training | Step 8 (`../stages/model-training.md`) |
| The test set has no rows for a failure the user sees in production | the test set cannot measure it | Step 5, with the user's approval, and with traces that show the failure (§ Changing the test set) |

Read the training data before blaming the student: download the Dataset the model trained on
(`../references/execution/cli.md` § Reading a Dataset) and look for the failure mode in the rows
the student learnt from.

## Step 4: Set Up the Next Iteration

Pick one hypothesis: the failure mode with the most errors that a single step can fix. That
step's stage lists its options in § What Can Be Changed to Improve the Next Iteration, each
with what it controls and how it moves the result. Choose the option whose effect matches the
failure mode, change one thing, and write the hypothesis and the change into the ledger before
submitting. Several training runs in parallel (students, epochs) still count as one hypothesis
about training.

Prefer the cheapest step that can fix the failure mode. From cheapest to most expensive:

- **Step 8, training**: nothing regenerates; one training credit.
- **Step 7, the train set**: a top-up on the best kept iteration's Dataset is the default (one
  generation credit and training); when the synthetic rows were bad, generation again on the
  parent of the bad run (the relabel-train Dataset, which keeps the relabelled rows); when the
  relabelled rows carry the mistake, relabelling again on the test-set Dataset, then
  generation.
- **Step 6, the teacher**: a teacher evaluation, then the train set and training again.
- **Step 5, the test set**: every earlier score becomes stale (§ Changing the test set).

**The train set grows by default.** Unless the synthetic data analysis or the training results
show that data was bad, the next train set is the best kept iteration's Dataset plus a top-up
(`../stages/build-a-train-set.md` § Topping Up).

### Changing the test set

The locked test set is in every Dataset after it, and the traces that could extend it are only
in the Datasets before the expands used them. So a change goes through an upload that keeps
the train rows:

- **Wrong reference answers**: download the best kept iteration's Dataset, correct the rows in
  `test.jsonl`, and upload the directory with `distil dataset create --data`
  (`../references/execution/cli.md` § The Dataset). The train rows come along; one
  `datasets_post` credit; the upload has no parent on the platform, so record the jump in the
  ledger.
- **Rows the test set lacks**: the same upload, with `traces.jsonl` replaced by traces that
  show the failure (new ones, or leftovers downloaded from the root Dataset), then
  `../stages/build-a-test-set.md` on it: relabelling into the test split, synthetic rows if
  needed, the review gate. Relabelling draws a fresh shuffled sample of the traces, so it adds
  rows; it never corrects existing ones.

Either way the test rows changed: start a new ledger section, re-run the teacher evaluations
(`build-a-model.md` Step 6), then train the best iteration's settings on the new Dataset to
get a comparable score before changing anything else. There is no way to score an existing
SLM on new rows.

## Step 5: Run

Re-enter `build-a-model.md` at the step chosen in this workflow's Step 4 and run it and every
step after it, up to and including training. Everything upstream is reused: the new run is an
override on an existing Dataset in the chain (`../references/platform.md` § Overrides), except
where Step 4 creates a new Dataset (a hand-corrected upload, or a new test set). Keep the
student of the best iteration so far, on the training stage's fast path, unless the hypothesis
is about the student.

New traces from the serving endpoint do not start an inner-loop iteration: they are the outer
loop's transition (`build-a-model.md` Step 10), which brings a new Dataset to Step 5.

When training finishes, fill in the ledger row: scores, the comparison with the previous
iteration, and whether the change is kept. A change that did not help is reverted: the next
iteration starts from the best iteration's settings and Dataset, not from the last one's. Then
go back to Step 1.

## Reporting

At every stop, and whenever the user asks, report from the ledger: the target, the best
iteration and its scores, what each iteration changed and what it did, and why the loop
stopped. When it stopped because the diagnosis needs the user, state the failure mode and the
decision needed. A deploy candidate goes back to `build-a-model.md` Step 9.
