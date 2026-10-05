# Stage: Teacher Evaluation

Runs the teacher on the full test set as a feasibility gate: if the teacher cannot solve the
task, nothing downstream will. There is no smoke run: the evaluation always runs on the full
test set (durations: `../references/platform.md` § Job status), so each iteration is a complete
evaluation.

## Working Directory

```
teacher-evaluation/
├── iteration-1/
│   ├── input/           # the override config and job description sent, if any
│   ├── run.md           # the SeedDataset id, the command and the TeacherEvaluation id
│   └── output/          # fetched metrics and predictions
└── iteration-2/         # next iteration: one change against iteration-1
```

## Step 1: Prepare the Input

The input is a SeedDataset id. A change to the teacher or the judge is a config or
job-description override on it (`../references/execution/cli.md` § Overrides), kept in `input/`.

The test split must be populated: an empty one stops the job
(`../references/data-preparation/overview.md` § Empty splits). The evaluation uses the
SeedDataset's train split for its few-shot examples. Only when there is no train split, set
`evaluation.num_few_shot_examples: 0`.

What to set deliberately:

- `base.teacher_model_name`: the model under evaluation, and the one synthetic data generation
  will use (`../references/model-catalog.md`). Choosing it is the point of this stage.
- `llm_as_a_judge_instructions`: what the judge accepts on free-text tasks
  (`../references/job-description.md`). Not valid for classification.
- `evaluation.num_few_shot_examples` (default 1): worked examples from the train split, at most
  the number of train rows (`../references/configuration.md` § Cross-field validation).
- `evaluation.llm_as_a_judge_model_name`: the judge model. It is set once and stays the same for
  every run, so scores stay comparable. When switching the teacher, keep the judge as it is
  (`../references/model-catalog.md` § Defaults).

## Step 2: Confirm the Setup with the User

Before submitting, present and confirm:

- the input data (row counts, task type, where it came from)
- the teacher under evaluation; never launch with a default the user did not see
- the judge setup for free-text tasks
- the remaining `teacher_evaluations_post` credits, one per iteration
  (`../references/platform.md` § Credits)

## Step 3: Run the Evaluation

Submit the evaluation (`../references/execution/cli.md` § Teacher evaluation) and record the
command and the TeacherEvaluation id in `run.md`.

## Step 4: Pull and Analyze the Results

Confirm the job succeeded, then fetch the metrics and the predictions into `output/`
(`../references/execution/cli.md` § Fetch metrics). Read the primary metric for the task
(`../references/evaluation-metrics.md` § Primary metric per task) and analyze:

1. **Judge check first**: sample predictions the judge marked bad. If they look correct, the
   judge is mismeasuring, and the judge instructions need fixing before anything else.
2. **Failure patterns**: group the incorrect predictions. One recurring pattern (one class always
   wrong, one format always missed) points at a targeted fix rather than at the teacher.

## Step 5: Verdict and Next Step

Review the score and the failure patterns with the user
(`../references/evaluation-metrics.md` § Verdicts):

- **PROCEED** when the user confirms this quality is acceptable in production: continue to
  `synthetic-data-generation.md`.
- **ITERATE**: try a better-suited teacher in a new iteration directory, or fix mislabelled test
  rows. Change the judge instructions only when Step 4 shows the judge is mismeasuring. The job
  description stays as it is (`../references/job-description.md` § What each field feeds).
- **RETHINK**: check the task type (`../references/task-types.md`), whether the task is well
  defined, and whether the judge measures the right thing. Fixing these usually means a new
  iteration of the whole loop (`../workflows/model-iterations.md`).

## What Can Be Changed to Improve the Next Iteration

- **The teacher** (`base.teacher_model_name`): it writes the training data, so its ceiling is
  the student's ceiling. A stronger or better-suited teacher raises what the student can reach.
  The judge stays as it is.
- **The judge instructions** (`llm_as_a_judge_instructions`): what counts as a correct answer on
  the free-text metrics. Change them only when Step 4 shows the judge is mismeasuring: every
  earlier score was measured with the old instructions and is not comparable. A stronger judge
  model (`evaluation.llm_as_a_judge_model_name`) is the other fix, with the same effect.

Cost: one `teacher_evaluations_post` credit. A new teacher also means synthetic data and
training again.
