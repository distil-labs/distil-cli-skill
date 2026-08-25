# Workflow: Dataset to Model

End-to-end model building starting from a labeled dataset. Each step is a stage file. Follow
the stage's own protocol and return here for the next step. Check in with the user at every
step boundary: each stage opens with its own setup gate, and no step's results are final
until the user has seen them.

Before Step 1, settle the execution backend: install the `distil` CLI per
`../references/execution/README.md` § Choose the backend, and record the result in `run.md`.

## Workflow Map

Print this map to the user when starting the workflow, before the first step:

```
1 Prepare Data ─── task type, job description, train/test
      │
      ▼
2 Teacher Evaluation ─── feasibility gate: PROCEED only
      │      (ITERATE: swap the teacher and re-run this step)
      ▼
3 Synthetic Data Generation ─── smoke ► full, with the passing teacher
      │
      ▼
4 Model Training ─── smoke ► full; fast or normal path
      │
      ▼
5 Decide ─┬─ close to teacher ──────────────► 6 Deploy
          ├─ above base, below teacher ─────► retune: back to Step 4 (cheap loop)
          └─ barely above base ─────────────► improving-a-model.md
```

## Step 1: Prepare the Data

Pick the task type (`../references/task-types.md`), write the job description
(`../references/job-description.md`), and build the input directory with train/test data
(`../references/data-preparation/overview.md` + task page). The job description must mirror
the user's production system prompt. It stays constant through everything that follows.
Present the prepared data to the user for review before moving on.

## Step 2: Teacher Evaluation

Run `../stages/teacher-evaluation.md`. Gate: only continue on PROCEED. On ITERATE or
RETHINK, work the stage's levers (teacher choice first) and re-run until the teacher passes.

With an empty test split this step cannot run and the gate is unavailable. That is the
exception, not a path to suggest (`../references/data-preparation/overview.md` § Empty
splits). Tell the user what they lose before continuing: the feasibility gate, the teacher
ceiling, and every number in Step 5. Then go to Step 3.

## Step 3: Synthetic Data Generation

Run `../stages/synthetic-data-generation.md` with the teacher that passed evaluation. The
stage's own smoke gate applies: only a generation setup whose sample passes analysis gets the
full run.

## Step 4: Model Training

Run `../stages/model-training.md` on the generated dataset (the stage asks the user about
credits and the student set).

## Step 5: Decide

Review the training results with the user and decide together. The gate is the fraction of the
base→teacher gap the student closed, `(tuned - base) / (teacher - base)`, and the branches are
owned by `../references/evaluation-metrics.md` § Verdicts:

- **`closed` ≥ 0.8 → Deploy**: for a sweep, pick the smallest student that clears the bar.
  Continue to Step 6.
- **`closed` 0.4-0.8 → Retune**: re-enter `../stages/model-training.md` from the same
  `base-input/` with a different student or tuning parameters. Nothing regenerates, so this is
  the cheap loop.
- **`closed` < 0.4 → Rerun**: the knowledge is not transferring and retuning will not fix it.
  Start a new iteration of the whole workflow (`improving-a-model.md`).

## Step 6: Deployment

Run `../stages/model-deployment.md` for the accepted model.

## Iterating on the Model

Improving a model is not starting over. It is a second iteration of this pipeline that
reuses the first iteration's artifacts. The recipe is `improving-a-model.md`: gap diagnosis,
optional test-set expansion, seed blending, targeted mutators.
