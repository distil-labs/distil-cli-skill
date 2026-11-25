# Operation: Teacher Evaluation

Runs one model as the teacher on every row of a Dataset's test split and scores its answers with
the project's metrics. The evaluation is zero-shot: the model answers each test row with its
system prompt and nothing added, so the train split is not read. It writes a TeacherEvaluation
with the scores and the per-row predictions; the Dataset is unchanged.

| | |
|---|---|
| **Called by** | the task-description phase (scoring a job description version, comparing teachers) and the test-set phase (the target, the noise, the production baseline) |
| **Price** | the tokens in the test set, at the evaluated model's price; a higher-effort thinking variant costs 2 or 4 times as much (`../references/platform.md` § Billing; every teacher's price: https://www.distillabs.ai/docs/getting-started/billing#teachers) |

## Settings that control the output

| Setting | File | What it changes |
|---|---|---|
| `base.teacher_model_name` | `config.yaml` | The model under evaluation: a candidate teacher, or the production model |
| `task_description` | `job_description.json` | The system prompt every test row is answered with. Overridden only when the calling phase is scoring a new version of it |
| `evaluation.llm_as_a_judge_model_name` | `config.yaml` | The judge. Fixed for the project: scores from two judges are not comparable |
| `llm_as_a_judge_instructions` | `job_description.json` | What the judge accepts (`../references/job-description.md`). Fixed for the project for the same reason |
| `base.variable_system_prompt` | `config.yaml` | Whether each row keeps its own system message (`../references/variable-system-prompts.md` § Evaluation) |

The judge field is resolved once, when the config is first expanded, so in a config read back
from the Dataset it already names the judge, and changing the teacher leaves it as it was
(`../references/model-catalog.md` § Defaults). Check it is still the agreed judge before each
submission.

There is no smoke run: each run is a complete evaluation on the full test split.

## Working directory

```
teacher-evaluation/
└── <name>/              # e.g. glm-5.3-jd-v2-run-1, production-gpt-4.1-mini
    ├── input/           # the config and job description sent
    ├── output/          # metrics.json, predictions.jsonl, job.log
    └── run.md           # Dataset id, model, change tested, price, TeacherEvaluation id, scores
```

A repeat of the same settings is a new directory (`<name>-run-2`): it is a new
TeacherEvaluation, and the calling phase reads the spread between repeats as the noise.

## Step 1: Prepare the input

The input is a Dataset id whose test split is populated; an empty test split stops the job
(`../references/data-preparation/overview.md` § Empty splits). Read the Dataset's config and job
description into `input/`, set `base.teacher_model_name` to the model under evaluation, and keep
the whole files: an override replaces each file whole (`../references/platform.md` § Overrides).

```bash
distil dataset download-metadata -d teacher-evaluation/<name>/input <dataset-id>   # free
# input/config.yaml → base.teacher_model_name: <model>
```

- **The production model** is named by its catalog entry in `provider.model` form
  (`../references/model-catalog.md` § Teacher models), not by the OpenRouter slug an endpoint
  uses. Find the matching entry. When there is none, the production model cannot be scored on
  the platform: report that to the calling phase.
- **Proprietary models** run in teacher evaluation only; they never write rows.
- **A new job description version** is a job-description override with only `task_description`
  changed; the judge fields stay as they are.
- **With `base.variable_system_prompt` on**, a second run with it set to `false` checks that the
  answers depend on each row's system prompt (`../references/variable-system-prompts.md`
  § Evaluation).

## Step 2: Price it

```bash
distil teacher-evaluation create-from-dataset --dry-run --config teacher-evaluation/<name>/input/config.yaml \
  --job-description teacher-evaluation/<name>/input/job_description.json <dataset-id>
```

Each model has its own price, so it differs between models on the same test set; the dry run
names the model each charge is priced as. Add the price to the run log's spending and to `run.md`. If the balance does not
cover it, the submission would be refused with 402: report both numbers to the user with
https://app.distillabs.ai/billing.

## Step 3: Submit

```bash
distil teacher-evaluation create-from-dataset --output json \
  --config teacher-evaluation/<name>/input/config.yaml \
  --job-description teacher-evaluation/<name>/input/job_description.json <dataset-id> | jq -r .id
```

Check the exit code (an error prints `{"error": …}`). Several evaluations run concurrently.
Record each TeacherEvaluation id in the run log's lineage, with the Dataset it ran on. The
config and job description of a running evaluation can be read back within seconds
(`distil teacher-evaluation download-metadata`), so an override can be checked before it ends.
To stop it before it finishes: `../references/execution/cli.md` § Cancel a job.

## Step 4: Wait and pull

Wait with `bash poll.sh teacher-evaluation <id>` (`../references/execution/cli.md` § Wait for a
job). On `JOB_FAILURE`, search the log for the first error; a failed job is refunded. On success,
pull the metrics, the predictions and the log (`../references/execution/cli.md` § Fetch metrics).
The predictions download writes to the current directory unless `--file-name` names a path:

```bash
distil teacher-evaluation metrics --output json <id> | jq .teacher_performance > teacher-evaluation/<name>/output/metrics.json
distil teacher-evaluation download-predictions --file-name teacher-evaluation/<name>/output/predictions.jsonl <id>
distil teacher-evaluation logs --output json <id> | jq -r .logs > teacher-evaluation/<name>/output/job.log
```

## Step 5: Read the output

**The score.** `teacher_performance` maps each metric name to a score from 0 to 1. Read the
project's primary metric from the run log header (`../references/evaluation-metrics.md`
§ Primary metric per task), or, when the project's metric is the user's own scorer, run it on the
predictions file. A metric the run did not compute is `null`, not absent. Record the
score in `run.md`, the run log's lineage row, and the reference points the calling phase uses.

**The predictions file** is JSONL, one test row per line, with `prompt` (the row's messages,
system message first), `completion` (the reference answer), `prediction` (the model's answer) and
that row's score under each metric name. Read one row and work from what is there. Keep the file:
the error analysis compares every later model against it.

**Judge check.** Take a sample of the rows the judge marked bad on the primary metric and read
each prediction against the task description. When they look correct, the judge is mismeasuring:
report it to the calling phase before any score is used. Once agreed, the judge and its rubric are
not changed, so a fix is the user's decision.

**Reference-based metrics and self-labelled test sets.** When the evaluated model also wrote the
test set's answers (by relabelling or generation), `llm-as-a-judge`, `binary` and `rouge` score it
against its own answers and flatter it; `llm-as-a-judge-reference-free` does not have that bias.
A prediction marked bad against the reference but good without it points at the reference row,
not the model (`../references/evaluation-metrics.md` § Metrics by task).
