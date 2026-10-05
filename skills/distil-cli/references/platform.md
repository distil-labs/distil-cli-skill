# The Platform

How the platform behaves. The commands are in `execution/cli.md`.

## Entities and jobs

Each stage produces one entity, and each entity is created either from files you supply or
by running a job over the entity before it. That chain is the pipeline:

```
PreparedTraces ─► PreparedTraces ─► SeedDataset ─┬─► TeacherEvaluation
 (uploaded)        (updated: test set built)          └─► TrainingDataset ─► SLM ─► Deployment
```

A PreparedTraces is uploaded or created from an inference endpoint's records. The test set
from traces job builds an updated PreparedTraces from it that carries a test set. A SeedDataset
comes from trace processing over a PreparedTraces, or directly from a prepared directory.
Everything downstream is created by naming the id of its parent, so every later command needs
the ids: record each one in the iteration's `run.md` as it is created.

An uploaded PreparedTraces and a directly created SeedDataset run no job: their status is
`JOB_SUCCESS` as soon as the create returns, and their logs are empty.

## Job status

| Status | Meaning |
|---|---|
| `JOB_NOT_STARTED` | accepted, not yet scheduled |
| `JOB_PENDING` | scheduled, waiting for capacity |
| `JOB_RUNNING` | running |
| `JOB_SUCCESS` | finished; outputs readable |
| `JOB_FAILURE` | failed |
| `JOB_STOPPED` | stopped deliberately |

Deployments report `deployment_status` and `endpoint_status` instead of `status`.

Creating a job returns its id at once, and only the status says how the job is going. How long
a job takes depends on the size of the data and the problem: a small dataset finishes in
minutes. Typical durations for a full-size run:

| Stage | Typical duration |
|---|---|
| Test set from traces | scales with `num_traces_to_relabel` and `num_synthetic_examples` |
| Trace processing | 45 min |
| Teacher evaluation | 30 min |
| Synthetic data generation | 90 min |
| Model training | 90 min |
| Deployment | 40 min |

So submit, then poll between other work and tell the user where the job is, rather than
blocking on one long wait.

When a job fails, its log holds the cause, but not always at the end. A crash often ends
with a second, unrelated error, so an out-of-memory failure can finish with a pickling error
thousands of characters after the `OutOfMemoryError` that caused it. Search the log for the
first error, rather than reading the last one.

A failed job produces no metrics, so no verdict is computable from it. Fix the cause and
submit again: the retry is a new entity from the same parent. Config and job description are
readable on some failed entities and not others, so record what you submitted rather than
relying on reading it back.

## Smoke runs

Synthetic data generation and training take a smoke flag that runs the job small, on the same
parent and with the same config otherwise:

- **Synthetic data generation smoke**: `synthgen.generation_target` is set to 128.
- **Training smoke**: one epoch (`tuning.num_train_epochs: 1`) on the 128 longest train rows
  and the 32 longest test rows of the TrainingDataset. The rows are chosen by length alone, not
  per class.

Both accept config and job-description overrides, and both spend their own credit routes
(§ Credits), never a full-run credit. Trace processing has no smoke flag: its smoke downloads
the traces, trims them to a subsample, and uploads that as its own PreparedTraces.

## Overrides

A job's `config.yaml` and `job_description.json` can each be varied at submission, so an
iteration is the same parent submitted again, not a new job input: `smoke-1`, `smoke-2` and
`full-1` all point at one SeedDataset.

Two rules govern this, and both fail silently:

- **Each file is replaced whole, never merged.** Anything you do not send takes the *library
  default*, not the parent's value, and that covers individual fields as well as whole
  sections. Sending only the section you are changing reverts everything else without an
  error. So every override starts by reading the parent's file, editing one field, and sending
  it back entire.
- **An override reaches config and job description and nothing else.** Changing the data means
  a new entity: a new upload for different traces, a new SeedDataset for different rows.

## Credits

Credits are metered per route, not from one pool, and a balance is a count of remaining
calls to that route. Reading the balance is free and answers at zero, so every stage's setup
gate quotes the balance for what it is about to spend.

| Stage | Route key | New account starts with |
|---|---|---|
| Trace upload | `prepared_traces_post` | 100 |
| Test set from traces | `prepared_traces_with_expanded_test_set_post` | 20 |
| Trace processing | `seed_datasets_from_prepared_traces_post` | 20 |
| Job input, also the validator | `seed_datasets_post` | 100 |
| Teacher evaluation | `teacher_evaluations_post` | 20 |
| Synthetic data generation, smoke | `training_datasets_from_seed_datasets_smoke_post` | 100 |
| Synthetic data generation, full | `training_datasets_from_seed_datasets_post` | 5 |
| Full TrainingDataset download | `training_datasets_download_get` | **0** |
| TrainingDataset from local files | `training_datasets_post` | 100 |
| Model training, smoke | `slms_from_training_datasets_smoke_post` | 100 |
| Model training, full | `slms_from_training_datasets_post` | 2 |
| Model re-upload | `slms_post` | **0** |
| Deployment | `deployments_from_slms_post` | 2 |
| Inference endpoint | `inference_endpoints_post` | 10 |

Two start at zero, so what they gate is unavailable until the user asks for a grant. Read a
balance against the whole plan rather than the next submission: N training runs need N
credits, and synthetic data generation is wasted if no training credit is left after it.

A call against an exhausted route is refused. A failed job spends its credit like any other.

Only metered routes appear in a balance. A route the platform never charges for is absent
rather than reported as unlimited, so a missing key is not a zero.

## What each stage produces

| Entity | Metrics | Files | Config + job description |
|---|---|---|---|
| PreparedTraces | original-model score, its predictions (**built from traces only**) | traces, test data | free |
| SeedDataset | none | train, test, unstructured | free |
| TeacherEvaluation | teacher score, its predictions | none | free |
| TrainingDataset | train and test size in bytes | **credit gated**; a free sample instead | free |
| SLM | base and tuned scores, the tuned model's predictions | model tarball | free; also the inference client |
| Deployment | none | none | none |

Metrics and files answer once that entity's own job reaches `JOB_SUCCESS`. Before that a
metrics field is null and a download names the job's state and fails. Config and job
description are settled at submission: a TeacherEvaluation or SLM answers within seconds of the
create, so an override can be read back from a running job. A TrainingDataset answers only once
its job succeeds.

Three conditions apply on top of that table:

- **Only a PreparedTraces built by the test set from traces job has metrics**, and only when
  that job ran with `traces_to_test_set.evaluate_original_model: true`. They hold the score of
  the model that produced the traces (the production model) on the relabelled test traces. An
  uploaded PreparedTraces runs no job, so its fields are null.
- **An SLM reports both scores but only the tuned model's predictions.** The base-vs-tuned gap
  is available as numbers. A base-model failure case is not.
- **The free TrainingDataset sample is capped.** It is at most 128 rows drawn from the training
  data and never contains test rows; take those from the parent SeedDataset, whose download is
  free. The training data holds the seed rows and the generated rows together, and nothing in a
  row says which kind it is. When the sample holds fewer than 128 rows it is the whole
  train split. Otherwise, the row count is an estimate: the byte size from the metrics divided by
  the mean row size in the sample. Reading every row needs the credit-gated download.

What the scores mean, and how to judge them: `evaluation-metrics.md`.
