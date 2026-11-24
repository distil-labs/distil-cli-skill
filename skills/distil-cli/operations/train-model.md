# Operation: Train a Model

Finetunes a student on a Dataset's train split, then evaluates both the base student and the
tuned student on its test split. It writes an SLM with the base and tuned scores, the tuned
model's per-row predictions, the model tarball and its inference client. The Dataset is
unchanged.

| | |
|---|---|
| **Called by** | the model-iteration phase, once per iteration (once per student when a range of students is compared) |
| **Price** | the train set's tokens times the epochs, at the student's tier (`../references/platform.md` § Billing). It grows with every iteration, because the train set grows. A smoke costs a fraction of that |
| **Time** | grows with the train set's tokens and the epochs. Estimate it from the time per epoch in the log of a full job already run in this project, scaled to the new train set. Before the first full job, give a rough estimate and say so: a smoke trains one epoch on the longest rows and does not predict the full job's time. Measure the effect of a training setting on time instead of assuming it |

## Settings that control the output

All are config overrides on the same Dataset; nothing is regenerated
(`../references/configuration.md` § base, § tuning).

| Setting | What it changes |
|---|---|
| `base.student_model_name` | The model trained (`../references/model-catalog.md`). A larger student has more capacity for the same data and costs more to serve; another family can suit a task better at the same size |
| `tuning.num_train_epochs` (default 4), `tuning.learning_rate`, `tuning.per_device_train_batch_size` (default 1) | The number and size of optimizer steps. More steps fit the training data more closely; at a fixed epoch count, a larger batch means fewer steps |
| `tuning.lora_r`, `tuning.lora_alpha_multiplier` | How much of the model training can change. A deployment accepts only `lora_r` of 8, 16, 32, 64, 128, 256, 320 or 512 |
| `tuning.max_completion_length` (default 2048) | How long an answer may be in evaluation before it is cut off; matters for long and reasoning answers. Training logs a warning when it is below the longest training completion |
| `tuning.memory_optimized_training`, `tuning.use_qlora` | What lets a larger student or batch fit in GPU memory (§ Dealing with OOM) |
| `base.should_expand_dataset` | Whether a multi-turn row becomes one example per assistant turn (`../references/configuration.md` § Conversation expansion) |
| `base.enable_thinking` | Train and evaluate with the student's thinking mode on (`../references/reasoning-models.md`) |

## Working directory

```
model-training/
├── smoke-data/               # the longest rows, uploaded as the smoke Dataset (one per Dataset trained on)
├── smoke-<student>-1/        # memory check; smoke-<student>-2 after an OOM fix
│   ├── input/  output/  run.md
└── <iteration>-<student>/    # the full run, e.g. it3-qwen3.5-4b
    ├── input/                # the config sent
    ├── output/               # metrics.json, predictions.jsonl, job.log
    └── run.md                # Dataset id, change tested, price, SLM id, scores
```

## Step 1: Prepare the input

The input is a Dataset id. Its config is readable only once its job reaches `JOB_SUCCESS`, so wait
for the expand that made it first. Read the config into `input/`, edit the fields this run varies,
and send the whole file: an override replaces it whole, and an omitted field takes the library
default (`../references/platform.md` § Overrides).

```bash
distil dataset download-metadata -d model-training/<iteration>-<student>/input <dataset-id>
```

An empty train split stops the run. An empty test split produces a model with no scores: `slm
metrics` returns nulls, which is not a failed job
(`../references/data-preparation/overview.md` § Empty splits).

## Step 2: Price it

Price the full run and, when a smoke is needed (Step 3), the smoke:

```bash
distil slm create-from-dataset --dry-run --config model-training/<iteration>-<student>/input/config.yaml <dataset-id>
```

Add each price to the run log's spending and to `run.md`. If the balance does not cover it, the
submission would be refused with 402: report both numbers to the user with
https://app.distillabs.ai/billing.

## Step 3: Smoke the memory

A smoke checks that the student fits in GPU memory before the full run is charged. Run it for a
student and memory setting that has not yet trained in this project, and again when the new rows
are longer than any trained before; skip it for a setup that already trained on similar rows.

Build the smoke Dataset once per Dataset trained on: the 128 longest train rows and the 32 longest
test rows, chosen by length alone, uploaded as a Dataset of their own (free). The longest rows set
peak memory (`../references/platform.md` § Smoke runs; commands in
`../references/execution/cli.md` § Model training). Then submit with the full run's config,
changed to `tuning.num_train_epochs: 1`, one series per student:

```bash
distil slm create-from-dataset --output json --config model-training/smoke-<student>-1/input/config.yaml <smoke-dataset-id> | jq -r .id
```

Wait for it (Step 5). It passes when the job succeeded, the training loss decreased in the log,
and base and tuned metrics came back. Do not download the model, and do not read the smoke's
scores as a quality signal. If it ran out of memory, apply the next setting in § Dealing with
OOM and smoke again.

## Step 4: Submit the full run

Submit on the Dataset from Step 1, never on the smoke Dataset, with the settings that passed the
smoke. One submission per student; they run concurrently
(`../references/execution/cli.md` § Model training):

```bash
distil slm create-from-dataset --output json --config model-training/<iteration>-<student>/input/config.yaml <dataset-id> | jq -r .id
```

Check the exit code. Record the SLM id in the run log's lineage and in the ledger row with its
price, and give the user https://app.distillabs.ai/slms/<slm-id>.
To stop it before it finishes: `../references/execution/cli.md` § Cancel a job.

## Step 5: Wait

Wait with `bash poll.sh slm <slm-id>` (`../references/execution/cli.md` § Wait for a job).
Training takes hours, so this is many runs of the loop: record the last status in `run.md` after
each one. `JOB_RUNNING` covers training and the evaluation after it, and the metrics appear only
at `JOB_SUCCESS`. On `JOB_FAILURE`, search the log for the first error, not the last: an
out-of-memory failure can end with an unrelated pickling error far after the `OutOfMemoryError`
(`../references/platform.md` § Job status). A failed job is refunded.

## Pull the results

```bash
out=model-training/<iteration>-<student>/output
distil slm metrics --output json <slm-id> \
  | jq '{base: .base_model_performance, tuned: .tuned_model_performance}' > $out/metrics.json
distil slm download-predictions --file-name $out/predictions.jsonl <slm-id>
distil slm logs --output json <slm-id> | jq -r .logs > $out/job.log
```

The downloads write to the current directory unless `--file-name` (or `-d`) names a path: the
predictions default to `<id>-<kind>-predictions.jsonl` there, and a download overwrites what is
there. Always pass the path into the run's `output/`
(`../references/execution/cli.md` § Output layout).

- `metrics.json` holds the base student's and the tuned student's scores, each a map from metric
  name to a score from 0 to 1. A metric the run did not compute is `null`, not absent.
- `predictions.jsonl` holds the **tuned** model's predictions only, one test row per line with
  `prompt`, `completion` (the reference), `prediction` and that row's score under each metric
  name. The base student's per-row predictions are not available.
- The model tarball (`distil slm download`) is needed only for local serving; the inference client
  alone comes with `distil slm download-metadata` (`../references/execution/cli.md` § Fetch model
  artifacts).

## Step 6: Read the output

Read the primary metric from the run log header for the base and the tuned student, or run the
project's own scorer on the predictions file when the metric is the user's. Record both in
`run.md` and fill the ledger row's scores. Every training run scores the base student again, so
the base score moves with the noise; the run log keeps one base score for the project: the
first training writes it into the header.

Check the log for `tuning.max_completion_length (X) is below the longest training completion
(Y)`: answers longer than the limit are cut off in evaluation and score near zero.

## Dealing with OOM

When a training run runs out of memory, apply these in order; each costs more speed or quality
than the one before:

1. Lower `tuning.per_device_train_batch_size` (halve it, down to 1).
2. `tuning.memory_optimized_training: 1` (gradient checkpointing; slower).
3. `tuning.memory_optimized_training: 2` (level 1 plus activation offloading to CPU RAM; much
   slower).
4. `tuning.use_qlora: true` together with `memory_optimized_training: 2` (4-bit base model,
   about a third of the GPU memory).
5. Remove the longest ~1% of train rows, which drive peak memory.

The first four are config overrides. The fifth changes the rows: download the Dataset, drop the
longest rows, upload the result, and record the counts in the run log (SKILL.md § Rules). A
smaller student is the other way out; that is the calling phase's choice.
