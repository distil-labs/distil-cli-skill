# CLI Command Reference

Complete reference for the Distil CLI. Every command works on one entity, addressed by its own ID.

## The Pipeline

Each stage produces an ID that the next stage consumes.

```
distil traces upload                        -> <traces-id>        (optional: traces path only)
distil upload create-from-traces            -> <upload-id>
  or distil upload create --data <dir>      -> <upload-id>
distil teacher-evaluation create-from-upload -> <teacher-evaluation-id>   (feasibility check)
distil training-dataset create-from-upload  -> <training-dataset-id>      (synthetic data gen)
distil slm create-from-training-dataset     -> <slm-id>                   (fine-tuning)
distil deployment create-from-slm           -> <deployment-id>            (hosted inference)
```

Every family follows the same shape: `create*` to make one, `list` / `show` to find it, `status` / `logs` to follow its job, `metrics` for results, `download*` for artifacts. Every family aliases `ls` to `list`. Every read command accepts `--output json` / `-o json`.

### Tracing the Chain

Entities are linked on the platform: **each one records its parent**, so you never have to keep a manual ledger of IDs. Given any ID you can recover the whole lineage behind it.

| Entity | Field pointing at its parent |
|--------|------------------------------|
| `deployment` | `slm_id` |
| `slm` | `training_dataset_id` (null when `source` is `direct_upload`) |
| `training_dataset` | `upload_id` (null when `source` is `direct_upload`) |
| `teacher_evaluation` | `upload_id` |
| `upload` | `prepared_traces_id` (null when `source` is `direct_upload`) |

Walking backwards from a deployment to the traces it came from:

```bash
slm=$(distil deployment show <deployment-id> --output json | jq -r '.slm_id')
td=$(distil slm show "$slm" --output json | jq -r '.training_dataset_id // empty')
upload=$(distil training-dataset show "$td" --output json | jq -r '.upload_id // empty')
traces=$(distil upload show "$upload" --output json | jq -r '.prepared_traces_id // empty')
```

Walking forwards, filter a `list` on the parent field:

```bash
# Teacher evaluations for one upload
distil teacher-evaluation list --output json | jq -r --arg u "<upload-id>" '.[] | select(.upload_id == $u) | .id'

# Training datasets generated from one upload
distil training-dataset list --output json | jq -r --arg u "<upload-id>" '.[] | select(.upload_id == $u) | .id'

# SLMs trained from one training dataset
distil slm list --output json | jq -r --arg d "<training-dataset-id>" '.[] | select(.training_dataset_id == $d) | .id'

# Deployments of one SLM
distil deployment list --output json | jq -r --arg s "<slm-id>" '.[] | select(.slm_id == $s) | .id'
```

The chain stops at anything created directly from local files — a `distil slm create` upload has no `training_dataset_id`, and a `distil training-dataset create` dataset has no `upload_id`, because no parent produced them. Use `// empty` or `// "none"` in jq so a null does not silently become the string `"null"`.

## Job Status and Polling

Async commands (upload, teacher evaluation, synthetic data generation, training, trace processing) return a job status. For the full status value list, the canonical polling loop, and the `--output json | jq` pattern, see `references/tasks/polling-jobs.md`.

**Summary:** Terminal values are `JOB_SUCCESS`, `JOB_FAILURE`, `JOB_STOPPED`. Anything else means in progress. Always extract status via `--output json | jq -r '.status'` — do not grep human-readable output.

## Authentication

### distil auth

Authenticate with the Distil Labs platform. Opens your browser to sign in, then hands the session back to the CLI. `distil login` is an alias.

```bash
distil auth
```

For headless environments (CI, no browser), pass credentials directly to skip the browser:

```bash
distil auth --email you@example.com --password "$DISTIL_PASSWORD"
# short flags: -e / -p
```

### distil signup

Open your browser to create a new Distil Labs account. Once you finish signing up you are logged in — no separate `distil auth` step is needed. Aliases: `distil register`, `distil join`.

```bash
distil signup
```

### distil whoami

Display the currently authenticated user.

```bash
distil whoami
```

### distil logout

Log out from the platform and clear credentials.

```bash
distil logout
```

## Prepared Traces

`distil traces` (alias `distil traces ls` for `list`, `distil traces create` for `upload`) manages sets of production traces. Prepared traces are the raw material: uploading them stores the files, and a separate step processes them into training and test data.

### distil traces upload

Store trace files as a prepared-traces resource and print its ID.

**Directory mode** -- expects standard filenames (`traces.jsonl`, `job_description.json`, `config.yaml`, and optionally `test.jsonl`) in the directory:

```bash
distil traces upload --data <directory>
# Output: Prepared traces created. ID: <traces-id>
```

**Individual file flags:**

```bash
distil traces upload \
  --traces <file> \
  --job-description <file> \
  --config <file> \
  [--test <file>]
```

| Flag | Required | Description |
|------|----------|-------------|
| `--data` | Yes* | Directory containing trace files (`traces.jsonl`, `job_description.json`, `config.yaml`). |
| `--traces` | Yes* | Path to traces file (`.jsonl`). |
| `--job-description` | Yes* | Path to job description file (`.json`). |
| `--config` | Yes* | Path to config file (`.yaml` or `.yml`). |
| `--test` | No | Path to a curated test data file (`.jsonl` only). |

\* Provide either `--data` or all three individual file flags (`--traces`, `--job-description`, `--config`), but not both.

This command only stores the files -- it does not start processing. Follow it with `distil upload create-from-traces <traces-id>`.

### distil traces list

```bash
distil traces list
distil traces list --output json
```

Newest first. Fetches all pages internally (up to 10,000 records).

```bash
# Most recent prepared-traces ID
distil traces list --output json | jq -r '.[0].id // "none"'
```

### distil traces show / status

```bash
distil traces show <traces-id>
distil traces status <traces-id>
```

Prepared traces are created synchronously, so `status` on one that exists always reports success. It is not a verdict on the traces themselves -- trace validity only surfaces when an upload is built from them, so read `distil upload status` for that.

### distil traces download

```bash
distil traces download <traces-id>
distil traces download <traces-id> --destination <directory>
```

Writes `traces.jsonl`, `job_description.json`, `config.yaml`, and `test.jsonl` (whichever exist) under the destination, using the filenames `distil traces upload --data` expects. Alias: `-d`.

### distil traces download-metadata

```bash
distil traces download-metadata <traces-id>
distil traces download-metadata <traces-id> --destination ./metadata    # -d also works
```

Writes only `config.yaml` and `job_description.json`, into `<traces-id>-metadata` by default. Use this instead of `download` when you want to read or edit the settings the traces were prepared with and do not need `traces.jsonl`. Every entity family has the same subcommand -- see `distil upload download-metadata`, `distil teacher-evaluation download-metadata`, `distil training-dataset download-metadata` and `distil slm download-metadata`.

## Uploads

`distil upload` (alias `distil uploads`, and `distil upload ls` for `list`) works with uploads by upload ID. An upload is a set of training and test data, whether it came from local files or from processing prepared traces. It is the entry point to the pipeline: teacher evaluation and synthetic data generation both read an upload.

All read commands accept `--output json` / `-o json`.

### distil upload create

Create an upload from local data files. Use either directory mode or individual file flags.

**Directory mode** -- expects standard filenames (`job_description.json`, `train.jsonl`, `test.jsonl`, `config.yaml`, and optionally `unstructured.jsonl`) in the directory:

```bash
distil upload create --data <directory>
# Output: Upload successful. Upload ID: <upload-id>
```

**Individual file flags:**

```bash
distil upload create \
  --job-description <file> \
  --train <file> \
  --test <file> \
  --config <file> \
  [--unstructured <file>]
```

| Flag | Required | Description |
|------|----------|-------------|
| `--data` | Yes* | Directory containing data files. |
| `--job-description` | Yes* | Path to job description file (`.json`). |
| `--train` | Yes* | Path to training data file (`.jsonl`). |
| `--test` | Yes* | Path to test data file (`.jsonl`). |
| `--config` | Yes* | Path to config file (`.yaml` or `.yml`). |
| `--unstructured` | No | Path to unstructured data file (`.jsonl`) for synthetic data generation. |

\* Provide either `--data` or the individual file flags (`--job-description`, `--train`, `--test`, `--config`), but not both.

`--config` is required. In directory mode the directory must contain `config.yaml` or `config.yml`. The command fails before uploading anything if it is missing, so a missing config is a fast, safe failure rather than a half-done upload.

### distil upload create-from-traces

Start a trace-processing job that turns prepared traces into an upload. This is the second half of the traces path.

```bash
distil upload create-from-traces <traces-id>
distil upload create-from-traces <traces-id> --config <file>
distil upload create-from-traces <traces-id> --job-description <file>
# Output: Processing started. Upload ID: <upload-id>
```

| Flag | Alias | Description |
|------|-------|-------------|
| `--config` | `-c` | Complete config file (`.yaml`/`.yml`) that replaces the prepared traces' own outright. |
| `--job-description` | | Job description file (`.json`) that replaces the prepared traces' own outright. |

Both are optional; omit them to reuse the prepared traces' own config and job description. Each **replaces** the corresponding file outright rather than merging into it, so a `--config` has to be complete: a partial config drops every key it leaves out. Pull the original with `distil traces download-metadata <traces-id>`, edit that, and pass it back.

Every `create-from-*` command except `distil deployment create-from-slm` takes the same two flags -- see `distil teacher-evaluation create-from-upload`, `distil training-dataset create-from-upload` and `distil slm create-from-training-dataset`. Combined with `download-metadata` this gives an edit-and-rerun loop: pull the metadata of a finished entity, edit the config, and pass it back with `--config` on the next `create-from-*`.

Each call produces a **new** upload, so re-running against the same `<traces-id>` with different parameters leaves earlier attempts intact for comparison. This is how you iterate on `trace_processing` params without re-uploading trace files.

### distil upload list

```bash
distil upload list
distil upload list --output json
```

Newest first. Fetches all pages internally (up to 10,000 records).

```bash
# Most recent upload ID
distil upload list --output json | jq -r '.[0].id // "none"'
```

### distil upload show / status / logs / metrics

```bash
distil upload show <upload-id>      # id, created_at, source, status
distil upload status <upload-id>    # status only -- use this when polling
distil upload logs <upload-id>      # logs of the job that produced the upload
distil upload metrics <upload-id>   # base model performance + predictions URL
```

`source` is `direct_upload` or `prepared_traces`. Uploads from local files are complete the moment they are created and have no logs; trace-derived uploads process asynchronously, so poll `status` until it reaches a terminal value (see `references/tasks/polling-jobs.md`) and read `logs` when one fails.

```bash
distil upload status <upload-id> --output json | jq -r '.status'
distil upload metrics <upload-id> --output json | jq '.base_model_performance'
```

`metrics` holds `base_model_performance` and `base_model_predictions_download_url`. Note the shape: `metrics` returns the metrics object directly, so the scores are at `.base_model_performance`, not nested under a `metrics` key.

### distil upload download

```bash
distil upload download <upload-id>
distil upload download <upload-id> --destination <directory>
```

Writes whichever of `train.jsonl`, `test.jsonl`, `unstructured.jsonl`, `config.yaml`, and `job_description.json` the upload has, using the filenames directory mode expects -- so the output feeds straight back into `distil upload create --data <directory>`.

Errors while the upload is still processing, so poll `distil upload status` first. Alias: `-d`. If one file fails to download the command reports that file and finishes the rest.

### distil upload download-metadata

```bash
distil upload download-metadata <upload-id>
distil upload download-metadata <upload-id> --destination ./metadata    # -d also works
```

Writes only `config.yaml` and `job_description.json`, into `<upload-id>-metadata` by default -- the settings the upload was created with, without the train and test data that `download` pulls.

### distil upload download-traces-predictions

```bash
distil upload download-traces-predictions <upload-id>
distil upload download-traces-predictions <upload-id> --file-name predictions.jsonl
```

Per-example predictions of the model that generated the traces, on the processed test set. Used to compare the original production model against the committee-relabeled ground truth. Default output filename: `<upload-id>-traces-predictions.jsonl`.

## Teacher Evaluations

`distil teacher-evaluation` (aliases `distil teacher-evaluations` and `distil teacher-eval`, and `distil teacher-evaluation ls` for `list`) works with teacher evaluations by their own ID. A teacher evaluation always belongs to exactly one upload.

A teacher evaluation is a feasibility check and performance benchmark: if the teacher model can solve your task, the student can learn it. Run it before spending credits on training.

All read commands accept `--output json` / `-o json`.

### distil teacher-evaluation create-from-upload

Start a teacher evaluation over an upload.

```bash
distil teacher-evaluation create-from-upload <upload-id>
distil teacher-evaluation create-from-upload <upload-id> --output json
distil teacher-evaluation create-from-upload <upload-id> --config <file>
distil teacher-evaluation create-from-upload <upload-id> --job-description <file>
# Output: Teacher evaluation started. Teacher Evaluation ID: <teacher-evaluation-id>
```

| Flag | Alias | Description |
|------|-------|-------------|
| `--config` | `-c` | Complete config file (`.yaml`/`.yml`) that replaces the upload's own outright. |
| `--job-description` | | Job description file (`.json`) that replaces the upload's own outright. |
| `--output json` | `-o json` | Emit the new teacher evaluation as JSON. |

Both overrides are optional; omit them to reuse the upload's own config and job description. A `--config` replaces the source's outright rather than merging into it, so it has to be complete; pair it with `download-metadata` to start from the original. Use this to re-evaluate one upload against a different teacher model without re-uploading the data.

The upload has to have finished processing first, so poll `distil upload status <upload-id>` until `JOB_SUCCESS`. Each call produces a **new** teacher evaluation, so earlier attempts stay intact for comparison.

### distil teacher-evaluation list

```bash
distil teacher-evaluation list
distil teacher-evaluation list --output json
```

Newest first. Fetches all pages internally (up to 10,000 records).

```bash
# Most recent teacher evaluation ID
distil teacher-evaluation list --output json | jq -r '.[0].id // "none"'

# Teacher evaluations for one upload
distil teacher-evaluation list --output json | jq -r --arg u "<upload-id>" '.[] | select(.upload_id == $u) | .id'
```

### distil teacher-evaluation show / status / logs / metrics

```bash
distil teacher-evaluation show <teacher-evaluation-id>      # id, created_at, upload_id, status
distil teacher-evaluation status <teacher-evaluation-id>    # status only -- use this when polling
distil teacher-evaluation logs <teacher-evaluation-id>      # logs of the evaluation job
distil teacher-evaluation metrics <teacher-evaluation-id>   # teacher performance + predictions URL
```

Teacher evaluations run asynchronously, so poll `status` until it reaches a terminal value (see `references/tasks/polling-jobs.md`) and read `logs` when one fails. `metrics` is empty until the job succeeds.

```bash
distil teacher-evaluation status <teacher-evaluation-id> --output json | jq -r '.status'
distil teacher-evaluation metrics <teacher-evaluation-id> --output json | jq '.teacher_performance'
```

### distil teacher-evaluation download-predictions

```bash
distil teacher-evaluation download-predictions <teacher-evaluation-id>
distil teacher-evaluation download-predictions <teacher-evaluation-id> --file-name predictions.jsonl
```

Per-example teacher predictions on the upload's test set. Used for analysis reports and identifying which examples the teacher gets right or wrong. Default output filename: `<teacher-evaluation-id>-teacher-evaluation-predictions.jsonl`.

### distil teacher-evaluation download-metadata

```bash
distil teacher-evaluation download-metadata <teacher-evaluation-id>
distil teacher-evaluation download-metadata <teacher-evaluation-id> --destination ./metadata    # -d also works
```

Writes `config.yaml` and `job_description.json` into `<teacher-evaluation-id>-metadata` by default. Other than its predictions, this is the only download a teacher evaluation has -- it is how you recover the config the evaluation ran with.

## Training Datasets

`distil training-dataset` (aliases `distil training-datasets` and `distil training-data`, and `distil training-dataset ls` for `list`) works with training datasets by their own ID.

A training dataset is an upload's data with synthetic training examples generated for it — the same four files an upload holds (`train.jsonl`, `test.jsonl`, `config.yaml`, `job_description.json`), with generated rows in `train.jsonl`. There is no unstructured data file.

**This is a required stage, not an optional inspection.** Training reads a training dataset, so generating one is how you get to `distil slm create-from-training-dataset`. The useful side effect is that you can look at the generated rows with `sample` before committing to the multi-hour training job.

All read commands accept `--output json` / `-o json`.

### distil training-dataset create-from-upload

Start a synthetic data generation job over an upload. Costs 2 credits.

```bash
distil training-dataset create-from-upload <upload-id>
distil training-dataset create-from-upload <upload-id> --output json
distil training-dataset create-from-upload <upload-id> --config <file>
distil training-dataset create-from-upload <upload-id> --job-description <file>
# Output: Synthetic data generation started. Training Dataset ID: <training-dataset-id>
```

| Flag | Alias | Description |
|------|-------|-------------|
| `--config` | `-c` | Complete config file (`.yaml`/`.yml`) that replaces the upload's own outright. |
| `--job-description` | | Job description file (`.json`) that replaces the upload's own outright. |
| `--output json` | `-o json` | Emit the new training dataset as JSON. |

Both overrides are optional; omit them to reuse the upload's own config and job description. A `--config` replaces the source's outright rather than merging into it, so it has to be complete; pair it with `download-metadata` to start from the original. Use `--config` to change generation settings between attempts without building a new upload.

The upload has to have finished processing first, so poll `distil upload status <upload-id>` until `JOB_SUCCESS`. Each call produces a **new** dataset, so earlier attempts stay intact for comparison.

### distil training-dataset create

Create a dataset directly from local files, with no generation job. Takes `--data <directory>`, or all four of `--train`, `--test`, `--config`, `--job-description`.

```bash
distil training-dataset create --data ./my-dataset-dir
distil training-dataset create --train train.jsonl --test test.jsonl --config config.yaml --job-description job_description.json
```

Because `download` writes the file names `create` reads, download → edit → create round-trips a dataset. This is the path for curating generated rows by hand before training on them.

### distil training-dataset list

```bash
distil training-dataset list
distil training-dataset list --output json
```

Newest first. Fetches all pages internally (up to 10,000 records).

```bash
# Most recent training dataset ID
distil training-dataset list --output json | jq -r '.[0].id // "none"'

# Datasets generated from one upload
distil training-dataset list --output json | jq -r --arg u "<upload-id>" '.[] | select(.upload_id == $u) | .id'

# Only the ones created by a generation job, not uploaded directly
distil training-dataset list --output json | jq -r '.[] | select(.source == "upload") | .id'
```

`source` is `upload` for a generated dataset and `direct_upload` for one created from local files.

### distil training-dataset show / status / logs / metrics

```bash
distil training-dataset show <training-dataset-id>      # id, created_at, source, upload_id, status
distil training-dataset status <training-dataset-id>    # status only -- use this when polling
distil training-dataset logs <training-dataset-id>      # logs of the generation job
distil training-dataset metrics <training-dataset-id>   # stored size of train.jsonl and test.jsonl
```

Generation runs asynchronously, so poll `status` until it reaches a terminal value (see `references/tasks/polling-jobs.md`) and read `logs` when one fails.

A directly created dataset has no job behind it: its `status` is `JOB_SUCCESS` as soon as it exists, and its `logs` are empty. That is normal, not a failure.

`metrics` reports object sizes, not job output — synthetic data generation writes no metrics artifact, unlike teacher evaluation and trace processing. Do not look here for quality numbers.

```bash
distil training-dataset status <training-dataset-id> --output json | jq -r '.status'
distil training-dataset metrics <training-dataset-id> --output json | jq '.train_data_size_bytes'
```

### distil training-dataset sample

Print up to 20 rows of the training data without downloading it. Free, and available for every dataset.

```bash
distil training-dataset sample <training-dataset-id>
distil training-dataset sample <training-dataset-id> --output json | jq '.rows[0]'
```

The sample is deterministic — seeded on the dataset ID, so repeated calls return the same rows. The 20-row cap is not configurable: it is what keeps a free endpoint from becoming an unmetered download. Prefer this over `download` when you only need to see what was generated.

### distil training-dataset download

```bash
distil training-dataset download <training-dataset-id>
distil training-dataset download <training-dataset-id> --destination ./dataset
```

Writes `train.jsonl`, `test.jsonl`, `config.yaml` and `job_description.json` into `<training-dataset-id>-data` unless `-d`/`--destination` says otherwise. **Credit-gated, with 0 credits granted by default** — expect a 402 unless credits have been granted for this route. Use `sample` when you only need to see the rows.

### distil training-dataset download-metadata

```bash
distil training-dataset download-metadata <training-dataset-id>
distil training-dataset download-metadata <training-dataset-id> --destination ./metadata    # -d also works
```

Writes only `config.yaml` and `job_description.json`, into `<training-dataset-id>-metadata` by default. Unlike `download` it fetches no generated rows, so prefer it when you only need the settings synthetic data generation ran with.

## SLMs

`distil slm` (alias `distil slms`, and `distil slm ls` for `list`) works with trained small language models by their own ID. An SLM is the model tarball plus the config that describes it, and it comes from one of two places: the platform trained it from a training dataset, or you uploaded it yourself.

All read commands accept `--output json` / `-o json`.

### distil slm create-from-training-dataset

Train an SLM from a training dataset. This is the distillation step: knowledge from the teacher is compressed into a compact student model.

```bash
distil slm create-from-training-dataset <training-dataset-id>
distil slm create-from-training-dataset <training-dataset-id> --output json
distil slm create-from-training-dataset <training-dataset-id> --config <file>
distil slm create-from-training-dataset <training-dataset-id> --job-description <file>
# Output: Training started. SLM ID: <slm-id>
```

| Flag | Alias | Description |
|------|-------|-------------|
| `--config` | `-c` | Complete config file (`.yaml`/`.yml`) that replaces the training dataset's own outright. |
| `--job-description` | | Job description file (`.json`) that replaces the training dataset's own outright. |
| `--output json` | `-o json` | Emit the new SLM as JSON. |

Both overrides are optional; omit them to reuse the training dataset's own config and job description. A `--config` replaces the source's outright rather than merging into it, so it has to be complete; pair it with `download-metadata` to start from the original. Use `--config` to retrain the same dataset with different training settings or a different student model.

Training typically takes several hours and burns credits that are hard to refund. Get the training dataset ID from `distil training-dataset create-from-upload` or `distil training-dataset list` (see `## Training Datasets` above).

### distil slm create

Upload an SLM you already have. Two forms, mutually exclusive:

```bash
# Directory mode -- expects model.tar and config.yaml (or config.yml)
distil slm create --data ./my-slm-dir

# Explicit paths -- both flags required together
distil slm create --model ./model.tar --config ./config.yaml
```

| Flag | Description |
|------|-------------|
| `--data` | Directory holding `model.tar` and `config.yaml`. |
| `--model` | Path to the model tarball (`.tar`). |
| `--config` | Path to the config (`.yaml` or `.yml`). |

The config cannot come from inside the tarball: a real `model.tar` holds `model/` and `model-adapter/` and nothing else. Uploading runs a job too, so poll `distil slm status <slm-id>` afterwards.

### distil slm list

```bash
distil slm list
distil slm list --output json
```

Newest first. Fetches all pages internally (up to 10,000 records).

```bash
# Most recent SLM ID
distil slm list --output json | jq -r '.[0].id // "none"'

# SLMs trained from one training dataset
distil slm list --output json | jq -r --arg d "<training-dataset-id>" '.[] | select(.training_dataset_id == $d) | .id'
```

### distil slm show / status / logs / metrics

```bash
distil slm show <slm-id>      # id, created_at, source, training_dataset_id, status
distil slm status <slm-id>    # status only -- use this when polling
distil slm logs <slm-id>      # logs of the job that produced the SLM
distil slm metrics <slm-id>   # base + tuned model performance, predictions URL
```

`source` is `training_dataset` for a trained SLM and `direct_upload` for one you uploaded.

Poll `status` until it reaches a terminal value (see `references/tasks/polling-jobs.md`) and read `logs` when one fails. `metrics` is empty until the job succeeds.

```bash
distil slm status <slm-id> --output json | jq -r '.status'
distil slm metrics <slm-id> --output json | jq '.tuned_model_performance'
distil slm metrics <slm-id> --output json | jq '.base_model_performance'
```

`metrics` is the one place that reports the base and the tuned student side by side, which is what you want when judging whether fine-tuning actually helped.

### distil slm download

```bash
distil slm download <slm-id>
distil slm download <slm-id> --destination ./my-slm    # -d also works
```

Writes `model.tar` and `config.yaml` into the destination directory, defaulting to `<slm-id>-slm/`. Those are the filenames `distil slm create --data` expects, so download and re-upload chain directly. Errors out while the job is still running, so poll `status` first.

This is also the starting point for serving the model yourself — see `## Local Serving` below.

### distil slm download-predictions

```bash
distil slm download-predictions <slm-id>
distil slm download-predictions <slm-id> --file-name predictions.jsonl
```

Per-example tuned-model predictions on the test set. Used for the training analysis report comparing tuned student vs. teacher and base student. Default output filename: `<slm-id>-slm-predictions.jsonl`.

### distil slm download-metadata

```bash
distil slm download-metadata <slm-id>
distil slm download-metadata <slm-id> --destination ./metadata    # -d also works
```

Writes `config.yaml`, `job_description.json` and `model_client.py`, into `<slm-id>-metadata` by default. Skips the `model.tar` that `download` pulls, so it is the cheap way to check what an SLM was trained with and to get the inference client without the weights.

## Deployments

`distil deployment` (alias `distil deployments`, and `distil deployment ls` for `list`) manages hosted inference deployments by their own ID. A deployment serves one SLM from Distil Labs infrastructure and exposes an OpenAI-compatible endpoint.

All read commands accept `--output json` / `-o json`.

### distil deployment create-from-slm

Deploy an SLM to the inference playground.

```bash
distil deployment create-from-slm <slm-id>
distil deployment create-from-slm <slm-id> --output json
# Output: Deployment started. Deployment ID: <deployment-id>
```

The SLM's job has to have succeeded first, so poll `distil slm status <slm-id>` until `JOB_SUCCESS`. A running deployment consumes inference credits — shut it down with `delete` when you are done.

### distil deployment list

```bash
distil deployment list
distil deployment list --output json
```

Newest first. Fetches all pages internally (up to 10,000 records).

```bash
# Most recent deployment ID
distil deployment list --output json | jq -r '.[0].id // "none"'

# Deployments of one SLM
distil deployment list --output json | jq -r --arg s "<slm-id>" '.[] | select(.slm_id == $s) | .id'
```

### distil deployment show / status / logs

```bash
distil deployment show <deployment-id>      # id, created_at, slm_id
distil deployment status <deployment-id>    # the show fields plus deployment_status and endpoint_status
distil deployment logs <deployment-id>      # logs of the deployment
```

`status` reports **two** status fields, and they are not the same:

| Field | Values | Meaning |
|-------|--------|---------|
| `deployment_status` | the job status values (`JOB_PENDING`, `JOB_RUNNING`, `JOB_SUCCESS`, …) | Whether the deploy itself finished. |
| `endpoint_status` | `running`, `stopped`, or `null` | Whether the endpoint is actually serving traffic. `null` until there is an endpoint at all. |

Poll on `endpoint_status` when you are waiting to send a request — a `JOB_SUCCESS` deploy whose endpoint is still `stopped` cannot answer yet.

```bash
distil deployment status <deployment-id> --output json | jq -r '.deployment_status'
distil deployment status <deployment-id> --output json | jq -r '.endpoint_status // "none"'
```

### distil deployment endpoint

Print the URL and API key for a deployment. This is how you get the credentials to send inference requests.

```bash
distil deployment endpoint <deployment-id>
distil deployment endpoint <deployment-id> --output json
```

Both `url` and `api_key` are `null` until the deployment is serving, so this is safe to poll without branching on readiness:

```bash
distil deployment endpoint <deployment-id> --output json | jq -r '.url // "not ready"'
distil deployment endpoint <deployment-id> --output json | jq -r '.api_key // "not ready"'
```

See `references/tasks/deployment-integration.md` for sending requests against the endpoint.

### distil deployment delete

Shut a deployment down and stop it consuming inference credits. Alias: `distil deployment shutdown`.

```bash
distil deployment delete <deployment-id>
distil deployment shutdown <deployment-id>
```

The SLM is untouched — only the serving infrastructure goes away. Deploy it again with `distil deployment create-from-slm <slm-id>`.

## Local Serving

There is no CLI-managed local server. To run an SLM on your own machine, download it and serve the artifacts yourself:

```bash
distil slm download <slm-id> --destination ./my-slm
# writes ./my-slm/model.tar and ./my-slm/config.yaml
tar -xf ./my-slm/model.tar -C ./my-slm
```

The tarball holds exactly two directories and nothing else:

| Path | Contents |
|------|----------|
| `model/` | The model weights, in Hugging Face format. |
| `model-adapter/` | The LoRA adapter. |

`vllm serve ./my-slm/model` works against this directly, since vLLM reads Hugging Face format. **llama-cpp does not** — it needs GGUF, and the SLM tarball ships no GGUF file, so serving with llama-cpp means converting first. See `references/tasks/deployment-integration.md` for the per-backend commands and for the hosted alternative.

## Utilities

### distil credits-balance

Report how many further calls the account may make to each metered endpoint. An endpoint appears only when calling it can be refused for want of credit; routes the platform never charges for are absent rather than listed as unlimited.

```bash
distil credits-balance
distil credits-balance --output json
```

Use this when a command fails with "Credit balance is too low" to see which route ran out, and before starting a long pipeline to check the stages ahead have credit. A balance of `0` on the route a stage needs means that stage will fail; the account needs a top-up before rerunning.

```bash
# Can this account still start a training run?
distil credits-balance --output json | jq -r '.balances.slms_from_training_datasets_post'
```

Route keys are named after the endpoint each stage calls, for example `prepared_traces_post` for `distil traces upload`, `uploads_post` for `distil upload create`, `training_datasets_from_uploads_post` for `distil training-dataset create-from-upload`, and `deployments_from_slms_post` for `distil deployment create-from-slm`.

### distil update

Update the Distil CLI to the latest version. The platform evolves quickly — run this before starting any new project to ensure new commands and features are available.

```bash
distil update
```

### distil docs

Open Distil Labs documentation in your default browser.

```bash
distil docs
```

## Global Options

These flags work with most commands:

| Flag | Description |
|------|-------------|
| `--output json` | Output results in JSON format for scripting and automation. Aliased to `-o json`. |
| `--help` | Display help information for any command. |
