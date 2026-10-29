# Stage: Model Iteration

Improves a trained model that is not good enough by running part of the pipeline again with one
change. The test set stays locked, so every score stays comparable, and the last model's
predictions say what to change.

This stage is one iteration, run with the user. `../workflows/autoresearch.md` runs it again
and again on its own, to a target agreed once.

## Iteration Map

Print this map to the user when the iteration starts:

```
 1 Read the scores
   ▼
 2 Inspect the predictions
   ▼
 3 Find the step the errors come from
   ▼
 4 Change one setting in that step
   ▼
 5 Run from that step through training ─────► back to 1
```

New traffic from the serving endpoint is not an iteration of this stage: it is the transition
of `../workflows/build-a-model.md` Step 10, which brings a new Dataset to the test set.

## The Ledger

`iterations.md` in the project root records every iteration. Read it before every decision: it
is what lets the work continue after a context reset. When it does not exist, create it: the
header from `test-set.md`, and row 1 for the first model, from `run.md`.

```markdown
Test set: dataset … (412 rows) · Teacher: … (0.81) · Production baseline: … (0.66)
Base score: 0.40 · Judge: … · Noise: ±0.02
Autoresearch: target 0.81 · budget $100, 8 iterations · spent $41.20

| Iteration | Hypothesis | Change | Re-ran from | Trained on | SLM | Primary metric | closed | vs best | Cost $ | Kept |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | first model | none | n/a | dataset … | slm … | 0.62 | 0.54 | n/a | 18.40 | yes |
| 2 | refund questions under-covered | mutator `topic` with 3 refund values, top-up of 2000 | train set | dataset … | slm … | 0.70 | 0.73 | +0.08 | 22.80 | yes |
```

- The header holds what every row is read against. The base score is one per project
  (`../references/evaluation-metrics.md` § Verdicts). The noise is the spread between two runs of
  the same evaluation: when the header has none, re-run the chosen teacher's evaluation
  (`teacher-evaluation.md`, the `<teacher-slug>-2` run) before reading any difference. The
  `Autoresearch:` line exists only under `../workflows/autoresearch.md`.
- Write a row when its run is submitted, with the price from `--dry-run`, and fill in the scores
  when training finishes: a row without scores is a run in flight.
- **vs best** is the primary metric minus the best kept row's. **Kept** is yes when that is
  more than the noise, otherwise no. The best kept row is where the next iteration starts.

When the test rows change, start a new section with a new header, because the scores above it
are stale.

## Step 1: Read the Scores

Read the latest model's scores on the project's primary metric next to the teacher, the base
student and the production model (`model-training.md` Step 7), and the previous iteration in
the ledger. The verdict comes from `closed` (`../references/evaluation-metrics.md`
§ Verdicts). A deploy candidate needs no iteration: say so and stop. Otherwise continue with
Step 2, a retune included: the predictions decide whether training is the right step.

A difference smaller than the noise in the ledger header is neither progress nor a regression.

## Step 2: Inspect the Predictions

Download the student's predictions (`../references/execution/cli.md` § Fetch metrics) and put
them next to the chosen teacher's predictions on the same test set: its TeacherEvaluation id is
in `test-set.md`. The teacher's predictions change only with the teacher or the test set, so
download them once into `teacher-evaluation/<teacher-slug>/output/` and reuse them. Match rows
on the last user message (with `base.variable_system_prompt` on, see
`../references/variable-system-prompts.md`): one test row can expand to several evaluation
rows, one per assistant turn. For each row the student got wrong, record what the student
answered, what the reference says, and whether the teacher got it right.

Group the errors into named failure modes: specific scenarios ("refunds for annual plans",
"dates in European format", "calls `search` when `lookup` was needed"), not "low score". Count
each mode. A mode with one example says little; a mode with many examples is a pattern.

## Step 3: Find the Step the Errors Come From

Map each failure mode to the cheapest step that can change it. The rows run from cheapest
(nothing regenerates) to most expensive (every earlier score becomes stale):

| What the predictions show | Likely cause | Re-run from |
|---|---|---|
| The teacher is right; the errors have no pattern and the gap to the teacher is small | capacity or too little training | training |
| The teacher is right; the student is wrong on a slice the training data barely covers | coverage | the train set: a top-up with `synthgen.mutators` for the slice |
| The teacher is right; the student is wrong the same way everywhere (format, a misread rule) | the training data teaches it wrong | the train set: `synthetic_data_generation_instructions` or a stronger teacher, or `trace_processing_instructions` when the relabelled rows carry the mistake |
| The teacher is wrong where the student is wrong | the teacher cannot do the task | the teacher |
| A baseline outside the platform (a lookup table, a rule, a public dataset) beats the teacher | the teacher shortlist is too small, not a reason to leave the platform | the teacher: more teachers from the catalog; never turn the outside data into train rows (`build-a-train-set.md` § Where New Rows Come From) |
| The judge marks correct answers bad, or wrong answers good | the measure is wrong | ask the user (`teacher-evaluation.md` § What Can Be Changed to Improve the Next Iteration) |
| The reference answer itself looks wrong, or the teacher and the student are wrong in the same way | wrong test labels | the test set (§ Changing the Test Set) |
| The test set has no rows for a failure the user sees in production | the test set cannot measure it | the test set (§ Changing the Test Set) |

When a failure mode may come from the training data, download the Dataset the model trained on
(`../references/execution/cli.md` § Reading a Dataset) and search its rows for that mode before
blaming the student.

## Step 4: Change One Setting

Pick one hypothesis: the failure mode with the most errors that a single step can change.
Change one thing, so the result shows what that thing did. Several training runs in parallel
(students, epochs) still count as one change to training. The options and what each one moves
are in each stage's § What Can Be Changed to Improve the Next Iteration:

- **Training** (`model-training.md`): on the best kept row's Dataset.
- **The train set** (`build-a-train-set.md`): a top-up on the best kept row's Dataset by
  default (§ Topping Up); that section and the stage's cheapest-first order say when to
  generate or relabel again instead, and on which Dataset.
- **The teacher**: the next-best teacher already scored in `test-set.md` first; new candidates
  through `teacher-evaluation.md`. Then the train set again, on the test-set Dataset.

`task_description` stays matched to the prompt the production system runs, so it changes only
when the user asks (`../references/job-description.md` § What each field feeds).

Confirm with the user: the failure modes, the hypothesis, the change, the smoke or fast path
for each stage it runs, and the price of every submission from `--dry-run`. This is the plan
the stages in Step 5 carry out (`../SKILL.md` § Stage Protocol).

### Changing the Test Set

The locked test set is in every Dataset after it, and the traces that could extend it are only
in the Datasets before the expands used them. So a change goes through an upload that keeps
the train rows, and it always needs the user's approval:

- **Wrong reference answers**: download the best kept row's Dataset, correct the rows in
  `test.jsonl`, and upload the directory with `distil dataset create --data`
  (`../references/execution/cli.md` § The Dataset). The train rows come along; the upload has
  no parent on the platform, so record the jump in the ledger.
- **Rows the test set lacks**: the same upload, with `traces.jsonl` replaced by traces that
  show the failure (new ones, or leftovers downloaded from the root Dataset), then
  `build-a-test-set.md` on it. Relabelling draws a fresh shuffled sample of the traces, so it
  adds rows; it never corrects existing ones.

Either way the test rows changed: start a new ledger section. Re-run the chosen teacher's and
the production model's evaluations, and train the best kept settings on the new Dataset at the
same time, for a comparable score before changing anything else. There is no way to score an
existing SLM on new rows.

## Step 5: Run

Run the stage chosen in Step 3 and every stage after it, through training: the teacher
(`teacher-evaluation.md`), then the train set (`build-a-train-set.md`), then training
(`model-training.md`). Each run is an override on a Dataset already in the chain
(`../references/platform.md` § Overrides), except where Step 4 uploads a new Dataset. Keep the
student of the best kept row unless the hypothesis is about the student.

When training finishes, fill in the ledger row. A change that did not help is reverted: the next
iteration starts from the best kept row's settings and Dataset, not from the last one's. Then go
back to Step 1.
