# Workflow: Model Iterations

Improving a trained model by running `build-a-model.md` again with changed settings, until the
model reaches an agreed target. An iteration starts at any entry point of `build-a-model.md` and
ends with a trained model and its evaluation results; the last model's predictions say what to
change. This workflow decides which stage to change. Each stage lists what can be changed in it,
in its § What Can Be Changed to Improve the Next Iteration.

## Workflow Map

Print this map to the user when starting the workflow, before the first step:

```
 0 Agree the target, the budget and the stop conditions with the user
   ▼
 1 Observe ─── scores of the latest model, against the teacher, the base student,
   │           the original model and the previous iteration, on one test set
   ▼  stop? ─── target reached / budget spent / plateau / needs the user ──► report
   ▼
 2 Inspect ─── download the student's and the teacher's predictions; group the errors
   ▼
 3 Diagnose ─── each failure mode → the earliest stage that can fix it
   ▼
 4 Set up ─── one hypothesis; changed settings for that stage; record it in the ledger
   ▼
 5 Run ─── re-enter build-a-model at that stage: entry point → synthetic data → training → eval
   └──────────► back to 1
```

## The Ledger

Keep `iterations.md` in the project root, one row per iteration, and read it before every
decision. It is what lets the loop continue after a context reset.

```markdown
| Iteration | Hypothesis | Change | Re-entered at | Entity ids | Primary metric | closed | vs previous | Kept |
|---|---|---|---|---|---|---|---|---|
| 1 | baseline | none | build-a-model Step 5 | seed …, td …, slm … | 0.71 | 0.52 | n/a | yes |
| 2 | refund questions under-covered | mutator `topic` with 3 refund values | Step 6 | td …, slm … | 0.78 | 0.69 | +0.07 | yes |
```

Each stage run also writes a new iteration directory in its own stage directory, as the stage
files describe. The ledger links them.

## Step 0: Agree the Target, the Budget and the Stop Conditions

Before the first autonomous iteration, agree with the user and write into the ledger's header:

- **The primary metric**: the task's default (`../references/evaluation-metrics.md` § Primary
  metric per task) unless the user names another. Every score in the ledger is this key.
- **The target**: the teacher's score, that is `closed` ≥ 1
  (`../references/evaluation-metrics.md` § Verdicts), unless the user specifies another one. A
  target the user gives takes precedence.
- **The budget**: the credits per route the loop may spend (`../references/platform.md`
  § Credits), and a maximum number of iterations. Every iteration spends at least one
  `slms_from_training_datasets_post` credit, so propose the iteration count from the balances.
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
- replacing the test set;
- changing the judge model or the judge instructions;
- spending beyond the budget, or on a route the budget does not cover;
- moving production traffic to a model.

Deploying a model only to re-test it (Step 4) is allowed within the budget.

## Step 1: Observe

Read the latest model's scores (`../stages/model-training.md` Step 7) on the task's primary
metric (`../references/evaluation-metrics.md` § Primary metric per task) and put them next to:

- the teacher's score on the same test set (the ceiling);
- the base student's score (the floor) and `closed`;
- the original model's score, when the test set came from traces (a floor the model must beat);
- the previous iteration's score, from the ledger.

Compare only scores measured on the same test set, and do not keep or revert a change on a
difference smaller than the noise measured in Step 0.

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

Map each failure mode to the earliest stage that can fix it:

| What the predictions show | Likely cause | Stage to re-enter |
|---|---|---|
| The reference answer itself looks wrong, or the teacher and the student are both wrong in the same way | wrong test labels | `../stages/test-set-from-traces.md` (relabelling), or the user corrects the rows |
| The teacher is wrong where the student is wrong | the teacher cannot do it | `../stages/teacher-evaluation.md` (another teacher), then synthetic data |
| The judge marks correct answers bad, or wrong answers good | the measure is wrong | ask the user: a new judge model or new judge instructions change what every score means, so every earlier score becomes stale, as with a new test set |
| The teacher is right; the student is wrong on a slice the training data barely covers | coverage | `../stages/synthetic-data-generation.md` |
| The teacher is right; the student is wrong the same way everywhere (format, a misread rule) | the training data teaches it wrong | `../stages/synthetic-data-generation.md`, or `../stages/trace-processing.md` when the seed rows carry the mistake |
| The teacher is right; the errors are spread with no pattern, and the gap to the teacher is small | capacity or too little training | `../stages/model-training.md` |
| The test set has no rows for a failure the user sees in production | the test set cannot measure it | `../stages/test-set-from-traces.md`, with the user's approval |

Read the training data before blaming the student: sample the TrainingDataset
(`../stages/synthetic-data-generation.md` Step 4) and look for the failure mode in the rows the
student learnt from.

## Step 4: Set Up the Next Iteration

Pick one hypothesis: the failure mode with the most errors that a single stage can fix. That
stage's § What Can Be Changed to Improve the Next Iteration lists its options, each with what it
controls and how it moves the result. Choose the option whose effect matches the failure mode,
change one thing, and write the hypothesis and the change into the ledger before submitting.
Several training runs in parallel (students, epochs) still count as one hypothesis about
training.

Prefer the cheapest stage that can fix the failure mode. From cheapest to most expensive:
training (nothing regenerates); synthetic data or a new teacher (a new teacher is checked by
teacher evaluation, then both regenerate the training data); trace processing (reprocesses the
seed data, then regenerates); the test set (every earlier score becomes stale).

**Run test set from traces again only when this workflow's Step 3 says the test set cannot
measure the failure.** A new test set replaces the old one, and every score in the ledger is
then stale: re-run teacher evaluation on it, and re-test the previous model on the new test rows.
To re-test, deploy it (`../stages/inference-endpoint.md` Steps 1-5), send each new test row
through its `model_client.py`, and score the answers against the references. Then delete the
deployment (`../stages/inference-endpoint.md` Step 8).

**The seed for synthetic data is, by default, the last iteration's training data.** Start fresh
only when the synthetic data analysis or the training results show that data was bad:

- **Default: reuse it.** The seed is the previous TrainingDataset's root `train.jsonl`, where
  seed and synthetic rows are already merged. Download it, pair it with the current test set
  and the unchanged job description, and create a new SeedDataset from the directory. Set
  `generation_target` to the top-up amount: it counts new examples only, and dedup runs against
  the seed, so the output is old and new merged. This spends a `training_datasets_download_get`
  credit, which starts at zero; at zero, start fresh. The download writes a directory that
  `seed-dataset create` accepts as it is:

  ```bash
  distil training-dataset download -d next-seed <training-dataset-id>
  # in next-seed/config.yaml: synthgen.generation_target (rows to add) and the new mutators
  distil seed-dataset create --data next-seed
  ```
- **Bad data → start fresh** from the same SeedDataset as before, with a full
  `generation_target`, after fixing what made the data bad.

## Step 5: Run

Re-enter `build-a-model.md` at the stage chosen in this workflow's Step 3 and run it and every
stage after it, up to and including training. Everything upstream is reused: the new run is an
override on the same parent entity (`../references/platform.md` § Overrides), except where
Step 4 creates a new entity (a reused-data SeedDataset, or a new test set). Keep the student of the best
iteration so far, on the training stage's fast path, unless the hypothesis is about the
student.

New traces from the serving endpoint can also start an iteration (`build-a-model.md` Step 9):
`build-a-model.md` Step 3 with the current `test.jsonl`, then `build-a-model.md` Step 5.

When training finishes, fill in the ledger row: scores, the comparison with the previous
iteration, and whether the change is kept. A change that did not help is reverted: the next
iteration starts from the best iteration's settings, not from the last one's. Then go back to
Step 1.

## Reporting

At every stop, and whenever the user asks, report from the ledger: the target, the best
iteration and its scores, what each iteration changed and what it did, and why the loop
stopped. When it stopped because the diagnosis needs the user, state the failure mode and the
decision needed.
