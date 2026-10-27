# The Platform

How the platform behaves. The commands are in `execution/cli.md`.

## Entities and jobs

Four entities carry the data, and each is created either from files you supply or by running a
job over the entity before it:

```
PreparedTraces ─► Dataset ─► Dataset ─► Dataset … ─┬─► TeacherEvaluation
 (traces.jsonl)    (traces,   (one expand            └─► SLM ─► Deployment
                   no rows)    operation each)
```

- A **PreparedTraces** holds one `traces.jsonl` and nothing else. It is uploaded, or created
  from an inference endpoint's records. It runs no job.
- A **Dataset** holds `config.yaml`, `job_description.json`, `train.jsonl`, `test.jsonl` and
  `traces.jsonl`; any data file can be empty. It is created in three ways, recorded
  in its `operation`:
  - `direct_upload`: the files on disk, any subset of the three data files. No job runs.
  - `from_prepared_traces`: a PreparedTraces plus a config and a job description. The traces
    are copied, train and test are empty. No job runs.
  - an **expand operation** on a parent Dataset, one of `relabel_traces_train`,
    `relabel_traces_test`, `generate_synthetic_data_train`, `generate_synthetic_data_test`. A
    job runs and writes a complete new Dataset; the parent is unchanged.
- A **TeacherEvaluation** and an **SLM** are created from any Dataset; the SLM is trained on
  its `train.jsonl` and scored on its `test.jsonl`.

A Dataset has at most one parent (`parent_prepared_traces_id` or `parent_dataset_id`), so a
project is a chain of Datasets, each one expand away from the previous. Every later command
needs the ids: record each one in the iteration's `run.md` as it is created.

An uploaded PreparedTraces, an uploaded Dataset and a Dataset from traces run no job: they exist
only once the create has succeeded, their status is `JOB_SUCCESS` as soon as the create
returns, and their logs are empty. A Dataset made by an expand has a job behind it, and its
status says how the job is going.

## The expand operations

Each operation reads one Dataset and writes a new one with rows added to one split:

| Operation | Reads from the parent | What the new Dataset adds |
|---|---|---|
| `relabel_traces_test` | `trace_processing.num_test_relabelled` traces | relabelled rows in `test.jsonl` |
| `relabel_traces_train` | `trace_processing.num_train_relabelled` traces | relabelled rows in `train.jsonl` |
| `generate_synthetic_data_test` | the `test.jsonl` rows as examples, traces as context | `synthgen.test_generation_target` synthetic rows in `test.jsonl` |
| `generate_synthetic_data_train` | the `train.jsonl` rows as examples, traces as context | `synthgen.train_generation_target` synthetic rows in `train.jsonl` |

Four rules apply to all of them:

- **Each expand removes the traces it used.** A trace is used at most once along a chain, so
  test and train never come from the same trace, and the trace pool shrinks with every expand.
- **Relabelling needs the traces it is asked for.** The job fails when the Dataset holds fewer
  distinct traces than `num_{split}_relabelled`. Fewer rows than traces can
  come out, because filtering and schema checks drop some; zero rows is still a success.
- **Generation uses traces as context when there are enough.** With
  `synthgen.use_traces_as_context: true` (the default) and a target of T rows, the job takes
  `T + min(T, 1000)` traces as context, or all that are left, and removes them. With fewer than
  `min(T / 4, 10)` traces it logs a warning, generates without context and leaves the traces
  alone. The split's existing rows are the examples; an empty split generates zero-shot.
- **The result is validated before it is written.** A new Dataset that breaks a dataset rule
  (`data-preparation/overview.md` § Validation rules) fails the expand.

Generation also drops new rows that duplicate rows already in the split
(`synthgen.validation_similarity_threshold`) or any row of the other split. The other split and
the config and job description are carried over with the same content. Relabelling writes
`traces.jsonl` back in `openai_messages` form without the system prompts, unless
`base.variable_system_prompt` is on (`variable-system-prompts.md`). Nothing in a row says whether it was
uploaded, relabelled or generated: to tell the new rows apart, compare
against the parent Dataset, whose download is free.

## Job status

| Status | Meaning | Terminal |
|---|---|---|
| `JOB_NOT_STARTED` | accepted, not yet scheduled | no |
| `JOB_PENDING` | scheduled, waiting for compute resources to be provisioned; logs are empty | no |
| `JOB_RUNNING` | running | no |
| `JOB_SUCCESS` | finished; outputs readable | yes |
| `JOB_FAILURE` | failed | yes |
| `JOB_STOPPED` | stopped deliberately | yes |

There is no `JOB_FAILED`. For training, `JOB_RUNNING` covers the training and the evaluation
after it, and the metrics appear only at `JOB_SUCCESS`.

Deployments report `deployment_status` and `endpoint_status` instead of `status`.

Creating a job returns its id at once, and only the status says how the job is going. How long
a job takes depends on the size of the data and the problem.

So submit, then wait with the one polling loop (`execution/cli.md` § Wait for a job), which
stops itself after a few minutes. Run it again between other work and tell the user where the
job is, rather than blocking on one long wait.

When a job fails, its log holds the cause, but not always at the end. A crash often ends
with a second, unrelated error, so an out-of-memory failure can finish with a pickling error
thousands of characters after the `OutOfMemoryError` that caused it. Search the log for the
first error, rather than reading the last one.

A failed job produces no output, so no Dataset rows, metrics or verdict come from it. Fix the
cause and submit again: the retry is a new entity from the same parent. Config and job
description are readable on some failed entities and not others, so record what you submitted
rather than relying on reading it back.

## Smoke runs

The four expand operations and training take a smoke flag that runs the job small, on the same
parent and with the same config otherwise:

- **Relabel traces smoke**: the split's `num_{split}_relabelled` is forced to 128.
- **Synthetic data generation smoke**: the split's generation target is forced to 128.
- **Training smoke**: one epoch (`tuning.num_train_epochs: 1`) on the 128 longest train rows
  and the 32 longest test rows of the Dataset. The rows are chosen by length alone, not per
  class.

All accept config and job-description overrides, and all are priced on their smaller inputs
(§ Billing), a fraction of the full run. A smoke expand writes a Dataset of its own, a side branch
of the parent: the full run is submitted on the same parent, not on the smoke's child, so the
smoke never costs the full run any traces.

Never continue from a smoke child: its config carries the smoke values. The full run, and every
expand after it, is submitted on the smoke's parent.

## Overrides

A job's `config.yaml` and `job_description.json` can each be varied at submission, so an
iteration is the same parent submitted again, not a new upload: `smoke-1`, `smoke-2` and
`full-1` all point at one parent Dataset.

Two rules govern this, and both fail silently:

- **Each file is replaced whole, never merged.** Anything you do not send takes the *library
  default*, not the parent's value, and that covers individual fields as well as whole
  sections. Sending only the section you are changing reverts everything else without an
  error. So every override starts by reading the parent's file, editing one field, and sending
  it back entire. A Dataset you uploaded returns its config as uploaded, with only the fields
  that were set; a Dataset made by an expand returns every field. To change a field that is
  absent, such as `synthgen.mutators`, add it to the file you read.
- **An override reaches config and job description and nothing else.** Changing the rows means
  a new Dataset: download the parent, edit the files, upload the directory.

An expand that ran with an override writes the overridden files into the new Dataset, so every
Dataset after it inherits the change.

## Billing

Jobs that run a model are paid from one prepaid balance in US dollars. Everything else is free:
uploads, traces objects, Datasets from traces, model uploads, inference endpoints, API keys,
downloads and reads. A billed job is priced from its data and config when it is submitted, and
the balance is charged then. The charge is the estimate, not what the job ends up using, and a
failed job is still charged.

| Job | Usage event | Quantity | Priced at the tier of |
|---|---|---|---|
| Teacher evaluation | `teacher_eval_tokens` | The tokens in the test set | The teacher |
| Relabel traces, smoke or full | `relabel_tokens` | The relabelled traces times their average tokens, once per teacher in the committee | Each relabelling teacher |
| Synthetic data generation, smoke or full | `synth_data_tokens` | The generation target times the average tokens of a row | The teacher |
| Model training, smoke or full | `sft_training_tokens`, `rl_training_tokens` | The train set's tokens times the epochs, split by `tuning.rlvr_dataset_size` | The student |
| Deployment | `deployment_created` | 1 | The student |

Every teacher and student has a price tier. Larger models and thinking variants are in the
higher tiers, so the teacher choice moves the price of every expand after it, and the student
choice the price of training and deployment.

Every stage's setup gate prices what it is about to submit with `--dry-run`
(`execution/cli.md` § Billing) and quotes the price and the balance. A dry run is free, starts
nothing, and takes the same config and job-description overrides as the real submission, so
price the exact run. Read the balance against the whole plan rather than the next submission:
a test set, a train set and a model take at least four full expand runs and a training run.
Training can only be priced once its train set exists, so before generating data, leave room
for it.

A submission the balance does not cover is refused with 402 and creates nothing:
`This job costs $12.40 and your balance doesn't cover it. Please top up to start it.` Never
submit to find out. When the price is over the balance, tell the user both numbers and ask them
to add funds on the billing page, https://app.distillabs.ai/billing. Give them the link: a
browser that `distil billing manage` opens is on the machine the agent runs on, which may not be
in front of the user.

## What each entity produces

| Entity | Metrics | Files | Config + job description |
|---|---|---|---|
| PreparedTraces | none | `traces.jsonl` | none: it holds neither |
| Dataset | train, test and traces size in bytes | all five files, free | free |
| TeacherEvaluation | teacher score, its predictions | none | free |
| SLM | base and tuned scores, the tuned model's predictions | model tarball | free; also the inference client |
| Deployment | none | none | none |

Metrics and files answer once that entity's own job reaches `JOB_SUCCESS`. Before that a
metrics field is null and a download names the job's state and fails. An empty split is an
empty file and reads `0` in the metrics. Config and job
description are settled at submission: a TeacherEvaluation or SLM answers within seconds of the
create, so an override can be read back from a running job. A Dataset made by an expand
answers only once its job succeeds.

Two conditions apply on top of that table:

- **No entity scores the production model on its own.** The baseline the student must beat is
  a TeacherEvaluation on the Dataset with the production model as `base.teacher_model_name`
  (`../stages/teacher-evaluation.md`).
- **An SLM reports both scores but only the tuned model's predictions.** The base-vs-tuned gap
  is available as numbers. A base-model failure case is not.

What the scores mean, and how to judge them: `evaluation-metrics.md`.
