# The Platform

How the platform behaves, independent of how you call it. The execution backends
(`execution/`) hold the commands and the request shapes; everything here is true of both.

## Entities and jobs

Each stage produces one **entity**, and each entity is created either from files you supply or
by running a job over the entity before it. That chain is the pipeline:

```
PreparedTraces ─► SeedDataset ─┬─► TeacherEvaluation
                               └─► TrainingDataset ─► SLM ─► Deployment
```

A SeedDataset is the job input. It comes either from trace processing or directly from a
prepared directory. Everything downstream is created by naming the id of its parent, so
**entity ids are the currency of the whole pipeline** — record each one in the iteration's
`run.md` as you go, because every later read resolves against it.

Supplying files is a separate step from creating the entity, and **a staged bundle expires
after seven days**. Files that never became an entity are deleted and must be supplied again;
once the entity exists it owns its own copy and is unaffected.

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

Runs take up to hours. Submit in the foreground — creating a job returns its id at once — and
poll in the background, writing to a log in the iteration directory you read between other
work. A creation that succeeds tells you nothing about the job; only the status does.

When a job fails, its log holds the cause — but not always at the end. A crash often unwinds
into a second, unrelated error, so an out-of-memory failure can finish with a pickling error
thousands of characters after the `OutOfMemoryError` that caused it. Search the log for the
first error, rather than reading the last one.

A failed job produces no metrics, so no verdict is computable from it and no gate applies. Fix
the cause and submit again: the entity is spent, and the retry is a new one from the same
parent. Config and job description are readable on some entities and not others once a job has
failed, so record what you submitted rather than relying on reading it back.

## Overrides

A job's `config.yaml` and `job_description.json` can each be varied at submission, so an
iteration is **the same parent submitted again**, not a new job input: `smoke-1`, `smoke-2` and
`full-1` all point at one SeedDataset.

Two rules govern this, and both bite quietly:

- **Each file is replaced whole, never merged.** Anything you do not send takes the *library
  default*, not the parent's value — individual fields as well as whole sections. Sending only
  the section you are changing silently reverts everything else. So every override starts by
  reading the parent's file, editing one field, and sending it back entire.
- **An override reaches config and job description and nothing else.** Changing the data means
  a new entity — which is why test-set expansion inverts train and test into a fresh
  SeedDataset rather than overriding the existing one.

## Credits

Credits are metered **per endpoint, not from one pool**, and a balance is a count of remaining
calls to that endpoint. Reading the balance is free and answers at zero, so every stage's setup
gate quotes the balance for what it is about to spend.

| Stage | Route key | New account starts with |
|---|---|---|
| Trace processing | `prepared_traces_post` | 100 |
| Trace processing | `seed_datasets_from_prepared_traces_post` | 20 |
| Job input — also the validator | `seed_datasets_post` | 100 |
| Teacher evaluation | `teacher_evaluations_post` | 20 |
| Synthetic data generation | `training_datasets_from_seed_datasets_post` | 5 |
| Training smoke calibration | `training_datasets_download_get` | **0** |
| Training smoke calibration | `training_datasets_post` | 100 |
| Model training | `slms_from_training_datasets_post` | 2 |
| Model re-upload | `slms_post` | **0** |
| Deployment | `deployments_from_slms_post` | **0** |

Three start at zero, so the stage they gate is unavailable until the user asks for a grant.
Read a balance against the whole plan rather than the next submission: N training runs need N
credits.

A call against an exhausted route is refused. A **failed job spends its credit** like any
other, but **invalid input costs nothing** — the balance is checked before validation and the
call recorded only once the entity exists, which is what makes creating a job input usable as
a free validator (`data-preparation/overview.md` § Validate before submitting).

Only metered routes appear in a balance. A route the platform never charges for is absent
rather than reported as unlimited.

## What each stage produces

| Entity | Metrics | Files | Config + job description |
|---|---|---|---|
| PreparedTraces | — | traces, test data | free |
| SeedDataset | original-model score, its predictions — **trace-derived only** | train, test, unstructured | free |
| TeacherEvaluation | teacher score, its predictions | — | free |
| TrainingDataset | train and test size in bytes | **credit gated**; a free sample instead | free |
| SLM | base and tuned scores, the tuned model's predictions | model tarball | free; also the inference client |
| Deployment | — | — | endpoint URL and key |

Metrics and files answer once that entity's own job reaches `JOB_SUCCESS`; before that they
are null. Config and job description fill earlier on some entities than others, so treat a
null as "not yet" and check the status to be sure.

Three conditions worth knowing before you plan around them:

- **A directly created SeedDataset has no metrics.** It runs no job — validation is
  synchronous — so there is nothing to report. The fields fill only on a trace-derived one,
  and only when trace processing ran with `evaluate_original_model: true`.
- **An SLM reports both scores but only the tuned model's predictions.** The base-vs-tuned gap
  is available as numbers; a base-model failure case is not.
- **The free TrainingDataset sample is capped**: a deterministic 128 rows at most, drawn from
  the first 384, train rows only. It is the whole dataset only when the dataset is smaller than
  that, and it never contains test rows — take those from the parent SeedDataset, whose
  download is free. Reading every row of a large dataset needs the credit-gated download.

What the scores mean, and how to judge them: `evaluation-metrics.md`.
