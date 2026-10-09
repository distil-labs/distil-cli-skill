# Stage: Teacher Evaluation

Runs a model on the full test set and scores it. The stage serves two purposes on the same
Dataset, and both run before the train set is built:

- **Pick the teacher.** A teacher evaluation per candidate, on the same test set, and the best
  one by the primary metric writes the train set. If no teacher can solve the task, nothing
  downstream will.
- **Score the production model.** A teacher evaluation with the model the user runs in
  production as `base.teacher_model_name`. Its score is the baseline the student must beat
  (`../references/evaluation-metrics.md` § Verdicts). It is not a candidate unless the user
  says so.

There is no smoke run: the evaluation always runs on the full test set (durations:
`../references/platform.md` § Job status), so each run is a complete evaluation.

## Working Directory

```
teacher-evaluation/
├── <teacher-slug>/
│   ├── input/           # the override config and job description sent
│   ├── run.md           # the Dataset id, the command and the TeacherEvaluation id
│   └── output/          # fetched metrics and predictions
├── <production-model-slug>/   # the baseline run
└── <teacher-slug>-2/    # the same teacher again: the noise measurement, or a judge fix
```

## Step 1: Prepare the Input

The input is the test-set Dataset id (`build-a-test-set.md` Step 5). Each run is a config
override on it (`../references/execution/cli.md` § Overrides), kept in that run's `input/`, with
`base.teacher_model_name` set to the model under evaluation.

The test split must be populated: an empty one stops the job
(`../references/data-preparation/overview.md` § Empty splits). The evaluation takes its
few-shot examples from the train split and caps `evaluation.num_few_shot_examples` to the rows
there are, so it runs zero-shot on a Dataset whose train split is still empty, which is the
normal case at this point.

What to set deliberately:

- **The shortlist**: the default teacher (`../references/model-catalog.md` § Defaults) and the
  production model, plus any teacher the user names. The production model is named by its
  catalog entry (`provider.model`, `../references/model-catalog.md` § Teacher models), not by
  the OpenRouter slug the endpoint uses; find the matching entry, and when there is none (most
  proprietary models), say so and skip the baseline: the user then judges the student on the
  absolute score.
- `llm_as_a_judge_instructions`: what the judge accepts on free-text tasks
  (`../references/job-description.md`). Not valid for classification.
- `evaluation.llm_as_a_judge_model_name`: the judge model. It is set once and stays the same for
  every run, so scores stay comparable. When switching the teacher, keep the judge as it is
  (`../references/model-catalog.md` § Defaults).

## Step 2: Confirm the Setup with the User

Before submitting, present and confirm:

- the test set (row count, where the rows came from)
- the shortlist, and which entry is the production model; never launch with a default the user
  did not see
- the judge setup for free-text tasks
- the remaining `teacher_evaluations_post` credits, one per run, against the whole shortlist,
  plus one more if the user plans to iterate: `../workflows/model-iterations.md` Step 0 repeats
  one run to measure the noise (`../references/platform.md` § Credits)

## Step 3: Run the Evaluations

Submit one evaluation per shortlist entry (`../references/execution/cli.md` § Teacher
evaluation); they run concurrently. Record each command and TeacherEvaluation id in its
`run.md`.

## Step 4: Pull and Analyze the Results

Confirm each job succeeded, then fetch the metrics and the predictions into `output/`
(`../references/execution/cli.md` § Fetch metrics). Put the scores on the primary metric
(`../references/evaluation-metrics.md` § Primary metric per task) in one table, and analyze the
best candidate's predictions:

1. **Judge check first**: sample predictions the judge marked bad. If they look correct, the
   judge is mismeasuring, and the judge instructions need fixing before anything else.
2. **Failure patterns**: group the incorrect predictions. One recurring pattern (one class always
   wrong, one format always missed) points at a targeted fix rather than at the teacher.
3. **Against the production model**: where the teacher beats it and where it does not. A
   teacher below the production model cannot write a train set that beats it. Read the
   reference-based metrics with care when the candidate is also the model that relabelled the
   test set: it is scored against its own answers, which flatters it; the reference-free judge
   metric does not have that bias.

## Step 5: Verdict and Next Step

Review the table and the failure patterns with the user
(`../references/evaluation-metrics.md` § Verdicts):

- **PROCEED** when the user confirms the best teacher's quality is acceptable in production:
  it becomes `base.teacher_model_name` for `build-a-train-set.md`. Record the chosen teacher,
  the production model's score (the baseline) and the TeacherEvaluation ids in the project
  root's `test-set.md`, next to the test-set Dataset id; `model-training.md` Step 7 and
  `../workflows/model-iterations.md` read them from there.
- **ITERATE**: add a better-suited teacher to the shortlist, or fix mislabelled test rows
  (`build-a-test-set.md` Step 4). Change the judge instructions only when Step 4 shows the judge
  is mismeasuring. The job description stays as it is (`../references/job-description.md`
  § What each field feeds).
- **RETHINK**: no teacher does the task well: check the task type
  (`../references/task-types.md`), whether the task is well defined, and whether the judge
  measures the right thing.

## What Can Be Changed to Improve the Next Iteration

- **The teacher** (`base.teacher_model_name`): it writes the training data, so its ceiling is
  the student's ceiling. A stronger or better-suited teacher raises what the student can reach.
  The judge stays as it is.
- **The judge instructions** (`llm_as_a_judge_instructions`): what counts as a correct answer on
  the free-text metrics. Change them only when Step 4 shows the judge is mismeasuring: every
  earlier score was measured with the old instructions and is not comparable. A stronger judge
  model (`evaluation.llm_as_a_judge_model_name`) is the other fix, with the same effect.

Cost: one `teacher_evaluations_post` credit per run. A new teacher also means the train set and
training again.
