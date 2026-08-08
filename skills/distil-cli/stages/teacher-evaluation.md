# Stage: Teacher Evaluation

Runs the teacher on the full test set as a feasibility gate: if the teacher cannot solve the
task, nothing downstream will, so its verdict decides whether to invest further. No smoke run
here: the evaluation is minutes-scale and always runs on the full test set, so each iteration
is a complete eval.

## Working Directory

```
teacher-evaluation/
├── iteration-1/
│   ├── input/           # the exact inputs this iteration ran
│   ├── run.md           # submission command and job identifiers
│   └── output/          # fetched metrics and predictions
└── iteration-2/         # next iteration: copy the previous input, change one thing
```

## Step 1: Prepare the Input Directory

The input is the standard directory (config.yaml, job_description.json, train/test.jsonl,
unstructured.jsonl where required). Prepare it following
`../references/data-preparation/overview.md` plus the task-specific page, and run the
validation checklist before submitting. If the data came from trace processing, the processed
output is already a valid input directory.

The evaluation is controlled by the `evaluation` config section
(`../references/configuration.md`) and the job description:

- `base.teacher_model_name` is the model under evaluation, and the one you are auditioning
  for synthetic data generation. Pick it deliberately from
  `../references/model-catalog.md` (tool-calling tasks restrict the choice). Trying a
  different teacher is the main reason to run another iteration of this stage.
- `llm_as_a_judge_instructions` in job_description.json matters most for free-text tasks: it
  defines what the judge accepts, and a vague one makes the whole verdict noisy
  (`../references/job-description.md`). Not valid for classification.
- `evaluation.num_few_shot_examples` (default 1) is how many worked examples the teacher is
  shown at evaluation time. It is unrelated to the per-class train-data floor in
  `../references/data-preparation/classification.md`, which is a minimum on the data you
  supply.
- `evaluation.llm_as_a_judge_model_name` picks the judge model. It defaults to
  `base.teacher_model_name`, which is usually what you want. But the default resolves when the
  config is first expanded, so a config read back for an override already names a judge. When
  you switch the teacher between iterations, switch this too, or the new teacher is judged by
  the old one.

## Step 2: Confirm the Setup with the User

Before submitting, present and confirm:

- the input data (row counts, task type, where it came from)
- the teacher under evaluation (`base.teacher_model_name`). Never launch with a silent
  default, since choosing the teacher is the point of this stage
- the judge setup (`llm_as_a_judge_instructions`, judge model) for free-text tasks
- the remaining `teacher_evaluations_post` credits, one per iteration
  (`../references/platform.md` § Credits)
- the plan: one full evaluation now, and what each verdict would mean next

## Step 3: Run the Evaluation

Submit the full evaluation via the execution backend (§ Submitting jobs) and record the
command and job identifiers in `run.md`.

## Step 4: Pull and Analyze the Results

Confirm the job succeeded, then pull the results into `output/` (the execution backend
§ Fetch metrics). There are two, and they arrive by different routes. The aggregated scores
come back inline as the `teacher_performance` object in the metrics response. The per-example
predictions are a separate file behind the presigned `predictions_download_url` alongside it.
Save the first as `metrics-eval-aggregated.json` yourself. The second is served under the name
`metrics-eval-full.jsonl`.

The predictions file is JSONL, one row per test example, with `prompt`, `completion`,
`prediction` and that example's own scores. Identify the primary metric for the task
(`../references/evaluation-metrics.md`) and analyze:

1. **Judge sanity check first**: sample predictions the judge marked bad. If they look
   correct, the score is lying, and the judge instructions need fixing before anything else.
2. **Failure patterns**: group the incorrect predictions. A single recurring pattern (one
   class always wrong, one format always missed) points at a targeted fix rather than a
   verdict problem.

## Step 5: Verdict and Next Step

Render the verdict using the relative gates in
`../references/evaluation-metrics.md` § Verdicts. Review the score with the user, because the
teacher is the quality ceiling everything downstream distills from:

- **PROCEED**: continue to synthetic data generation (`synthetic-data-generation.md`).
- **ITERATE**: the primary lever is the teacher model. This stage is where you pick the right
  teacher for this problem, so try a better-suited one (`../references/model-catalog.md`) in
  a new iteration directory. Fixing mislabeled or ambiguous test examples is also fair game.
  Do NOT tune the job description to chase the score: it must stay compliant with the system
  prompt the user runs in production and remain constant across iterations. Only adjust
  judge instructions when Step 4 shows the judge is mismeasuring.
- **RETHINK**: step back before touching levers. Is the task type right
  (`../references/task-types.md`), is the task well-defined, is the judge measuring the right
  thing? Fixing these usually means a fresh iteration of the whole workflow
  (`../workflows/improving-a-model.md`).
