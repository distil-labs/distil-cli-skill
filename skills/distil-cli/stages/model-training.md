# Stage: Model Training

Finetunes the student on the Dataset's train split and evaluates the base and the tuned
student on its test split, which gives the base-vs-tuned-vs-teacher comparison. Training is the
GPU stage, so how much to experiment here is the user's budget decision.

An empty train split stops the run, and an empty test split produces a model with no scores
(`../references/data-preparation/overview.md` § Empty splits). With no test set, tell the user
before submitting that the model cannot be measured yet.

## Working Directory

```
model-training/
├── base-input/          # the Dataset id and its config; every run derives from it
│                        # (one per Dataset trained on: base-input-2/ for the next iteration)
├── smoke-<student>-1/   # memory check; one series per selected student
│   ├── input/           # the config sent for this run
│   ├── run.md           # the Dataset id, the override sent and the SLM id returned
│   └── output/          # fetched metrics and training logs
├── smoke-<student>-2/   # next memory check for that student, after an OOM fix
└── full-<student>-1/    # one full training per student
```

## Step 1: Prepare the Input

The input is a Dataset id: the one `build-a-train-set.md` produced, or an earlier one to
retrain on (`../references/execution/cli.md` § Model training, § Reuse the data for
training-only runs). Its config is readable only once its job reaches `JOB_SUCCESS`, so poll
the last expand to completion before reading it.

Read its config once into `base-input/` (`../references/execution/cli.md` § Overrides). Every run
edits the fields it varies and sends the whole file as an override.

The settings that matter are in `base` and `tuning` (`../references/configuration.md`):
`base.student_model_name`, `tuning.per_device_train_batch_size` (default 1) and
`tuning.num_train_epochs` (default 4). Multi-turn rows follow `base.should_expand_dataset`
(`../references/configuration.md` § Conversation expansion). For a reasoning student, see
`../references/reasoning-models.md`.

## Step 2: Confirm the Setup with the User

Before submitting anything, present and confirm:

- **The path**, from the balances (`../references/platform.md` § Credits):
  - `slms_from_datasets_smoke_post` pays for the memory-check smokes (Steps 3-5), one per
    attempt. With none left, go to Step 6.
  - `slms_from_datasets_post` pays for one full run each, so a sweep of N students needs N.
  - With credits for both, the user chooses: normal (a memory check per student, then the full
    run) or fast (skip Steps 3-5, the usual choice for a re-run of a proven setup).
- **The student list**, fixed now, because each student is checked for memory separately. Fast
  path: one student, the best iteration's student when there is one, else `Qwen3.5-4B` unless
  the deployment target needs smaller (`../references/model-catalog.md` § Defaults). Normal path: students across the sizes that fit
  the deployment target (`../references/model-catalog.md` § Size tiers).

## Step 3: Smoke Run

A smoke checks that a student fits in memory, on a subsample of the data, before the full run
spends a training credit. Submit with `--smoke` (`../references/execution/cli.md` § Model
training; what it trains on: `../references/platform.md` § Smoke runs) and an override that sets
the student and the batch size the full run will use. One `smoke-<student>-N` series per
student, since students of different sizes fit differently. Record `run.md`.

## Step 4: Pull and Analyze the Smoke Outputs

Confirm the job succeeded: the run fit in memory, the training loss decreased in the log, and
base and tuned metrics came back. Do not download the model, and do not read the smoke's scores
as a quality signal.

## Step 5: Iterate Until the Smoke Passes

If the smoke ran out of memory, apply the next setting in § Dealing with OOM and smoke again.
Move on once every student fits.

## Step 6: Full Run

Present the plan first: the students, each one's settings, and the cost (one
`slms_from_datasets_post` credit per student). On the user's go-ahead, submit one training per
student against the same Dataset, without `--smoke`: all the training data, `num_train_epochs`
at its default of 4, and the settings that passed the smoke. Each submission sends a full copy
of the config with its student and settings. The submissions run concurrently.

## Step 7: Analyze the Results

Confirm each job succeeded, then fetch the base and tuned metrics into `output/`
(`../references/execution/cli.md` § Fetch metrics). With no test set, report that there are no
scores and stop. Otherwise compare on the primary metric agreed for the project
(`../references/evaluation-metrics.md` § Primary metric per task): base student (floor), the
production model (the baseline, from `test-set.md`; `teacher-evaluation.md` Step 5), teacher
(ceiling), tuned student, one row per student. What to
do next is decided in `../workflows/build-a-model.md` Step 8.

## Dealing with OOM

When a training run runs out of memory, apply these in order; each costs more speed or quality
than the one before:

1. Lower `per_device_train_batch_size` (halve it, down to 1).
2. `memory_optimized_training: true` (activation offloading and gradient checkpointing; much
   slower).
3. `use_qlora: true` together with `memory_optimized_training: true` (4-bit base model, about 3x
   less GPU memory).
4. Remove the longest ~1% of training rows, which drive peak memory. Download the Dataset,
   remove the rows from `train.jsonl`, and upload the directory as a new Dataset with
   `distil dataset create --data` (`../references/execution/cli.md` § Supplying files; one
   `datasets_post` credit).

The first three are config overrides; only the fourth changes the data.

## What Can Be Changed to Improve the Next Iteration

- **The student** (`base.student_model_name`, `../references/model-catalog.md`): a larger student
  has more capacity to learn the same data, at a higher serving cost. A different family can
  suit a task better at the same size.
- **How much it trains** (`tuning.num_train_epochs`, `tuning.learning_rate`,
  `tuning.per_device_train_batch_size`): the number and size of the optimizer steps. More steps
  fit the training data more closely; at a fixed epoch count, a larger batch means fewer steps.
- **The LoRA capacity** (`tuning.lora_r`, `tuning.lora_alpha_multiplier`): how much of the model
  training can change. Deployment accepts only some `lora_r` values
  (`../references/configuration.md` § tuning).
- **Evaluation length** (`tuning.max_completion_length`): how long an answer may be before it is
  cut off in evaluation, which matters for long and reasoning answers.
- **Memory settings** (§ Dealing with OOM): what lets a larger student or batch fit at all.

All are config overrides on the same Dataset, and nothing regenerates, so this is the least
expensive stage to iterate on. Cost: one `slms_from_datasets_post` credit per run; several runs
can go in parallel.
