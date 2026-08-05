# CLI Command Reference

Complete reference for the Distil CLI. Command aliases: `distil model` = `distil models` = `distil m` — all three work identically.

## Job Status and Polling

Async commands (upload, teacher evaluation, training, trace processing) return a job status. For the full status value list, the canonical polling loop, and the `--output json | jq` pattern, see `references/tasks/polling-jobs.md`.

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

## Model Management

### distil model create

Create a new model with the specified name. Returns the model ID used in all subsequent commands.

```bash
distil model create <name>
```

- `<name>` -- A human-readable name for your model (e.g., `customer-support-classifier`).

### distil model list

List all your models with their IDs, names, and status.

```bash
distil model list
distil model list --output json
```

### distil model show

Show detailed information about a specific model, including all component IDs (upload IDs, teacher evaluation IDs, training info).

```bash
distil model show <model-id>
distil model show <model-id> --output json
```

**JSON schema** (from `--output json`):

```json
{
  "id": "4cd3f76d-ffab-4244-a70e-54092df485b0",
  "name": "review-schema-probe",
  "created_at": "2026-04-17T22:28:09.549401Z",
  "training": null,
  "upload_ids": [],
  "teacher_evaluation_ids": [],
  "prepared_traces_ids": [],
  "training_status": "JOB_NOT_STARTED",
  "evaluation_results": null,
  "training_evaluation_results": null,
  "task_details": null
}
```

Field notes:

| Field | Description |
|-------|-------------|
| `id` | Model UUID. |
| `name` | Human-readable name set at `distil model create`. |
| `created_at` | ISO 8601 timestamp (UTC). |
| `training` | `null` until training starts; otherwise an object with training job details. |
| `upload_ids` | Array of upload UUIDs, **latest first**. Element 0 is the current upload. |
| `teacher_evaluation_ids` | Array of teacher evaluation UUIDs, latest first. |
| `prepared_traces_ids` | Array of prepared-traces UUIDs, latest first. Legacy field — list prepared traces with `distil traces list` instead. |
| `training_status` | One of the job status values (see `references/tasks/polling-jobs.md`). |
| `evaluation_results` | Teacher evaluation aggregate metrics once complete. |
| `training_evaluation_results` | Training aggregate metrics once complete. |
| `task_details` | Task-specific details populated after upload. |

**Common jq queries:**

```bash
# Latest upload ID (used to verify a re-upload actually took)
distil model show <model-id> --output json | jq -r '.upload_ids[0] // "none"'

# Latest teacher evaluation ID
distil model show <model-id> --output json | jq -r '.teacher_evaluation_ids[0] // "none"'

# Current training status
distil model show <model-id> --output json | jq -r '.training_status'
```

## Data Upload

### distil model upload-data

Upload training data for a model. Use either directory mode or individual file flags.

**Directory mode** -- expects standard filenames (`job_description.json`, `train.jsonl`, `test.jsonl`, `config.yaml`) in the directory:

```bash
distil model upload-data <model-id> --data <directory>
```

**Individual file flags:**

```bash
distil model upload-data <model-id> \
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

### distil model upload-traces / reprocess-traces (removed)

Both commands have been removed -- their API endpoint no longer exists. Each now prints the replacement steps and exits **non-zero**, so a script that calls them fails rather than silently doing nothing.

Replace `upload-traces` with `distil traces upload` followed by `distil upload create-from-traces`. Replace `reprocess-traces` with `distil upload create-from-traces <traces-id> --config <file>`. See `## Prepared Traces` and `## Uploads` below.

### distil model download-data

Download the uploaded data files for a model.

```bash
distil model download-data <model-id>
```

### distil model download-traces-predictions

Download per-example predictions of the original production model after trace processing completes. Used to compare the original model against the committee-relabeled ground truth.

```bash
distil model download-traces-predictions <model-id>
distil model download-traces-predictions <model-id> --file-name predictions.jsonl
```

Default output filename: `<model-id>-traces-predictions.jsonl`.

Takes a model ID and reads the data uploaded with `upload-data`. To download by upload ID instead, use `distil upload download-traces-predictions <upload-id>`.

### distil model download-teacher-evaluation-predictions

Download per-example teacher model predictions on the test set after teacher evaluation completes. Used for analysis reports and identifying which examples the teacher gets right or wrong.

```bash
distil model download-teacher-evaluation-predictions <model-id>
distil model download-teacher-evaluation-predictions <model-id> --file-name teacher-predictions.jsonl
```

### distil model download-training-predictions

Download per-example tuned student model predictions on the test set after training completes. Used for the training analysis report comparing tuned student vs. teacher and base student.

```bash
distil model download-training-predictions <model-id>
distil model download-training-predictions <model-id> --file-name student-predictions.jsonl
```

### distil model upload-status

Show the status of the data uploaded with `upload-data`, plus the base model metrics when there are any.

```bash
distil model upload-status <model-id>
distil model upload-status <model-id> --output json
distil model upload-status <model-id> --logs
```

| Flag | Alias | Description |
|------|-------|-------------|
| `--logs` | `-l` | Also fetch the logs of the job that produced the upload. |
| `--output json` | `-o json` | Emit `{"status": …, "metrics": {…}}`, plus a `logs` key when `--logs` is set. |

The JSON shape is worth noting when polling: `status` is the job status string, and `metrics` holds `base_model_performance` and `base_model_predictions_download_url`.

```bash
distil model upload-status <model-id> --output json | jq -r '.status'
```

To check an upload by its own ID, use `distil upload status <upload-id>` (see `## Uploads`).

## Prepared Traces

`distil traces` (alias `distil traces ls` for `list`, `distil traces create` for `upload`) manages sets of production traces. Prepared traces are the raw material: uploading them stores the files, and a separate step processes them into training and test data.

### distil traces upload

Store trace files as a prepared-traces resource and print its ID. Takes no model ID.

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
distil traces download <traces-id> --data-destination <directory>
```

Writes `traces.jsonl`, `job_description.json`, `config.yaml`, and `test.jsonl` (whichever exist) under the destination, using the filenames `distil traces upload --data` expects. Alias: `-d`.

## Uploads

`distil upload` (alias `distil uploads`, and `distil upload ls` for `list`) works with uploads by upload ID. An upload is a set of training and test data, whether it came from local files or from processing prepared traces.

All read commands accept `--output json` / `-o json`.

### distil upload create

Create an upload from local data files. Same flags as `distil model upload-data`, minus the model ID -- including the required `--config`.

```bash
distil upload create --data <directory>
```

### distil upload create-from-traces

Start a trace-processing job that turns prepared traces into an upload. This is the second half of the traces path, and the replacement for the removed `reprocess-traces`.

```bash
distil upload create-from-traces <traces-id>
distil upload create-from-traces <traces-id> --config <file>
distil upload create-from-traces <traces-id> --job-description <file>
# Output: Processing started. Upload ID: <upload-id>
```

| Flag | Alias | Description |
|------|-------|-------------|
| `--config` | `-c` | Config file (`.yaml`/`.yml`) merged over the prepared traces' own config on top-level keys. |
| `--job-description` | | Job description file (`.json`) that replaces the prepared traces' own outright. |

Both are optional; omit them to reuse the prepared traces' own config and job description. The merge is per top-level key, so passing a config containing only `trace_processing` keeps the rest of the original config intact.

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
```

### distil upload download

```bash
distil upload download <upload-id>
distil upload download <upload-id> --data-destination <directory>
```

Writes whichever of `train.jsonl`, `test.jsonl`, `unstructured.jsonl`, `config.yaml`, and `job_description.json` the upload has, using the filenames directory mode expects -- so the output feeds straight back into `distil model upload-data --data <directory>` or `distil upload create --data <directory>`.

Errors while the upload is still processing, so poll `distil upload status` first. Alias: `-d`. If one file fails to download the command reports that file and finishes the rest.

### distil upload download-traces-predictions

```bash
distil upload download-traces-predictions <upload-id>
distil upload download-traces-predictions <upload-id> --file-name predictions.jsonl
```

Per-example predictions of the model that generated the traces, on the processed test set. Default output filename: `<upload-id>-traces-predictions.jsonl`.

## Teacher Evaluation

### distil model run-teacher-evaluation

Start a teacher evaluation to validate that a large model can solve your task. This is a feasibility check and performance benchmark before training.

```bash
distil model run-teacher-evaluation <model-id>
```

Runs against the data uploaded with `upload-data`. To evaluate an upload by its own ID, use `distil teacher-evaluation create-from-upload` (see `## Teacher Evaluations`).

### distil model teacher-evaluation

Check the status and results of the teacher evaluation.

```bash
distil model teacher-evaluation <model-id>
distil model teacher-evaluation <model-id> --output json
distil model teacher-evaluation <model-id> --logs
```

| Flag | Alias | Description |
|------|-------|-------------|
| `--logs` | `-l` | Also fetch the logs of the evaluation job. |
| `--output json` | `-o json` | Emit `{"status": …, "metrics": {…}}`, plus a `logs` key when `--logs` is set. |

The JSON shape mirrors `distil model upload-status`: `status` is the job status string, and `metrics` holds `teacher_performance` and `predictions_download_url`.

```bash
distil model teacher-evaluation <model-id> --output json | jq -r '.status'
distil model teacher-evaluation <model-id> --output json | jq '.metrics.teacher_performance'
```

## Teacher Evaluations

`distil teacher-evaluation` (aliases `distil teacher-evaluations` and `distil teacher-eval`, and `distil teacher-evaluation ls` for `list`) works with teacher evaluations by their own ID. A teacher evaluation always belongs to exactly one upload.

All read commands accept `--output json` / `-o json`.

### distil teacher-evaluation create-from-upload

Start a teacher evaluation over an upload. This is the only creation path that does not go through a model.

```bash
distil teacher-evaluation create-from-upload <upload-id>
distil teacher-evaluation create-from-upload <upload-id> --output json
# Output: Teacher evaluation started. Teacher Evaluation ID: <teacher-evaluation-id>
```

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

Per-example teacher predictions on the upload's test set. Default output filename: `<teacher-evaluation-id>-teacher-evaluation-predictions.jsonl`.

## Training Datasets

`distil training-dataset` (aliases `distil training-datasets` and `distil training-data`, and `distil training-dataset ls` for `list`) works with training datasets by their own ID.

A training dataset is an upload's data with synthetic training examples generated for it — the same four files an upload holds (`train.jsonl`, `test.jsonl`, `config.yaml`, `job_description.json`), with generated rows in `train.jsonl`. There is no unstructured data file.

Training already generates this data internally, so this family is for **inspecting** what would be trained on, or for curating it. `distil model run-training` still runs from the upload; pointing training at a dataset ID is not supported.

All read commands accept `--output json` / `-o json`.

### distil training-dataset create-from-upload

Start a synthetic data generation job over an upload. Costs 2 credits.

```bash
distil training-dataset create-from-upload <upload-id>
distil training-dataset create-from-upload <upload-id> --output json
# Output: Synthetic data generation started. Training Dataset ID: <training-dataset-id>
```

The upload has to have finished processing first, so poll `distil upload status <upload-id>` until `JOB_SUCCESS`. Each call produces a **new** dataset, so earlier attempts stay intact for comparison.

### distil training-dataset create

Create a dataset directly from local files, with no generation job. Takes `--data <directory>`, or all four of `--train`, `--test`, `--config`, `--job-description`.

```bash
distil training-dataset create --data ./my-dataset-dir
distil training-dataset create --train train.jsonl --test test.jsonl --config config.yaml --job-description job_description.json
```

Because `download` writes the file names `create` reads, download → edit → create round-trips a dataset.

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
distil training-dataset download <training-dataset-id> --data-destination ./dataset
```

Writes `train.jsonl`, `test.jsonl`, `config.yaml` and `job_description.json` into `<training-dataset-id>-data` unless `-d`/`--data-destination` says otherwise. **Credit-gated, with 0 credits granted by default** — expect a 402 unless credits have been granted for this route.

## Training

### distil model run-training

Start training to distill knowledge from the teacher into a compact student model.

```bash
distil model run-training <model-id>
```

Training typically takes several hours. See "Job Status Values" at the top of this file for the full list of statuses and how to check them reliably.

### distil model training

Check the status and results of the training job. After training completes, this also shows evaluation metrics comparing the SLM against the teacher.

```bash
distil model training <model-id>
```

To work with the trained model by its own ID instead of through a model, see `## SLMs`.

## SLMs

`distil slm` (alias `distil slms`, and `distil slm ls` for `list`) works with trained small language models by their own ID. An SLM is the model tarball plus the config that describes it, and it comes from one of two places: the platform trained it from a training dataset, or you uploaded it yourself.

All read commands accept `--output json` / `-o json`.

### distil slm create-from-training-dataset

Train an SLM from a training dataset.

```bash
distil slm create-from-training-dataset <training-dataset-id>
distil slm create-from-training-dataset <training-dataset-id> --output json
# Output: Training started. SLM ID: <slm-id>
```

Training takes several hours. Get the training dataset ID from `distil training-dataset create-from-upload` or `distil training-dataset list` (see `## Training Datasets` above).

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

Poll `status` until it reaches a terminal value (see `references/tasks/polling-jobs.md`) and read `logs` when one fails. `metrics` is empty until the job succeeds.

```bash
distil slm status <slm-id> --output json | jq -r '.status'
distil slm metrics <slm-id> --output json | jq '.tuned_model_performance'
```

`metrics` is the one place that reports the base and the tuned student side by side, which is what you want when judging whether fine-tuning actually helped.

### distil slm download

```bash
distil slm download <slm-id>
distil slm download <slm-id> --destination ./my-slm    # -d also works
```

Writes `model.tar` and `config.yaml` into the destination directory, defaulting to `<slm-id>-slm/`. Those are the filenames `distil slm create --data` expects, so download and re-upload chain directly. Errors out while the job is still running, so poll `status` first.

### distil slm download-predictions

```bash
distil slm download-predictions <slm-id>
distil slm download-predictions <slm-id> --file-name predictions.jsonl
```

Per-example tuned-model predictions on the test set. Default output filename: `<slm-id>-slm-predictions.jsonl`.

## Retuning

### distil model retune

Retune an existing model with new tuning parameters. Creates a new model based on a previously trained one.

**Using a tuning parameters file:**

```bash
distil model retune <model-id> \
  --name <name> \
  --student-model <model> \
  --tuning-parameters <file>
```

**Using a full config file** (only the `tuning` section is used):

```bash
distil model retune <model-id> \
  --name <name> \
  --student-model <model> \
  --config <file>
```

| Flag | Alias | Required | Description |
|------|-------|----------|-------------|
| `--name` | `-n` | Yes | Name of the new retuned model to be created. |
| `--student-model` | `-s` | Yes | Student model to use for retuning. |
| `--tuning-parameters` | `-t` | Yes* | Path to tuning parameters file (`.json` or `.yaml`). |
| `--config` | `-c` | Yes* | Path to config file (`.json` or `.yaml`) -- only the `tuning` section is used. |

\* Provide either `--tuning-parameters` or `--config`, but not both.

## Retuning

### distil model retune

Retune an existing model with new tuning parameters. Creates a new model based on a previously trained one.

**Using a tuning parameters file:**

```bash
distil model retune <model-id> \
  --name <name> \
  --student-model <model> \
  --tuning-parameters <file>
```

**Using a full config file** (only the `tuning` section is used):

```bash
distil model retune <model-id> \
  --name <name> \
  --student-model <model> \
  --config <file>
```

| Flag | Alias | Required | Description |
|------|-------|----------|-------------|
| `--name` | `-n` | Yes | Name of the new retuned model to be created. |
| `--student-model` | `-s` | Yes | Student model to use for retuning. |
| `--tuning-parameters` | `-t` | Yes* | Path to tuning parameters file (`.json` or `.yaml`). |
| `--config` | `-c` | Yes* | Path to config file (`.json` or `.yaml`) -- only the `tuning` section is used. |

\* Provide either `--tuning-parameters` or `--config`, but not both.

## Deployment

### distil model deploy local

Deploy a model locally using llama-cpp as the inference backend (experimental). Requires llama-cpp installed on your machine.

```bash
distil model deploy local <model-id>
distil model deploy local --port 9000 <model-id>
distil model deploy local --port 9000 --logs <model-id>
```

| Flag | Description |
|------|-------------|
| `--port <port>` | Port number for local llama-server (default: 8000). |
| `--logs` | Show llama-server logs during local deployment. |
| `--output json` | Output results in JSON format. |

### distil model deploy remote

Deploy a model to Distil Labs hosted inference infrastructure.

```bash
distil model deploy remote <model-id>
distil model deploy remote --client-script <model-id>
```

| Flag | Description |
|------|-------------|
| `--client-script` | Output only the client script for the deployment. |
| `--output json` | Output results in JSON format. |

### distil model deploy remote --deactivate

Deactivate a remote deployment to conserve credits.

```bash
distil model deploy remote --deactivate <model-id>
```

### distil model invoke

Get the command to query a deployed model. Outputs a ready-to-run `uv run` command pointing to a client script.

```bash
distil model invoke <model-id>
```

## Model Download

### distil model download

Download your trained model files.

```bash
distil model download <model-id>
```

## Utilities

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
| `--output json` | Output results in JSON format for scripting and automation. |
| `--help` | Display help information for any command. |
