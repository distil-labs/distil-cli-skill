# CLI Command Reference

Complete reference for the Distil CLI. Command aliases: `distil model` = `distil models` = `distil m` — all three work identically.

## Job Status and Polling

Async commands (upload, teacher evaluation, training, trace processing) return a job status. For the full status value list, the canonical polling loop, and the `--output json | jq` pattern, see `references/tasks/polling-jobs.md`.

**Summary:** Terminal values are `JOB_SUCCESS`, `JOB_FAILURE`, `JOB_STOPPED`. Anything else means in progress. Always extract status via `--output json | jq -r '.status'` — do not grep human-readable output.

## Authentication

### distil login

Authenticate with the Distil Labs platform. Opens a browser for login.

```bash
distil login
```

### distil register

Create a new Distil Labs account.

```bash
distil register
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
| `prepared_traces_ids` | Array of prepared-traces UUIDs, latest first. Populated when `upload-traces` is used. |
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

**Directory mode** -- expects standard filenames (`job_description.json`, `train.csv`, `test.csv`, `config.yaml`) in the directory:

```bash
distil model upload-data <model-id> --data <directory>
```

**Individual file flags:**

```bash
distil model upload-data <model-id> \
  --job-description <file> \
  --train <file> \
  --test <file> \
  [--config <file>] \
  [--unstructured <file>]
```

| Flag | Required | Description |
|------|----------|-------------|
| `--data` | Yes* | Directory containing data files. |
| `--job-description` | Yes* | Path to job description file (`.json`). |
| `--train` | Yes* | Path to training data file (`.csv` or `.jsonl`). |
| `--test` | Yes* | Path to test data file (`.csv` or `.jsonl`). |
| `--config` | No | Path to config file (`.yaml` or `.json`). |
| `--unstructured` | No | Path to unstructured data file (`.csv`) for synthetic data generation. |

\* Provide either `--data` or the individual file flags (`--job-description`, `--train`, `--test`), but not both.

### distil model upload-traces

Upload production traces for a model. Traces are an alternative to structured data uploads. The command uploads traces and processes them into training and test data in one step.

**Directory mode** -- expects standard filenames (`traces.jsonl`, `job_description.json`, `config.json`/`config.yaml`) in the directory:

```bash
distil model upload-traces <model-id> --data <directory>
```

**Individual file flags:**

```bash
distil model upload-traces <model-id> \
  --traces <file> \
  --job-description <file> \
  --config <file> \
  [--test <file>]
```

| Flag | Required | Description |
|------|----------|-------------|
| `--data` | Yes* | Directory containing trace files (`traces.jsonl`, `job_description.json`, `config.json`/`config.yaml`). |
| `--traces` | Yes* | Path to traces file (`.jsonl`). |
| `--job-description` | Yes* | Path to job description file (`.json`). |
| `--config` | Yes* | Path to config file (`.json` or `.yaml`). |
| `--test` | No | Path to a curated test data file (`.jsonl` or `.csv`). |

\* Provide either `--data` or all three individual file flags (`--traces`, `--job-description`, `--config`), but not both.

### distil model reprocess-traces

Reprocess previously uploaded traces with a new trace processing config. Uses the most recently uploaded prepared traces for the model. You must have already uploaded traces with `upload-traces` first.

```bash
distil model reprocess-traces <model-id> --trace-processing-config <file>
```

You can also pass a full config file -- only the `trace_processing` section will be used:

```bash
distil model reprocess-traces <model-id> --config <file>
```

| Flag | Alias | Required | Description |
|------|-------|----------|-------------|
| `--trace-processing-config` | `-t` | Yes* | Path to trace processing config file (`.json` or `.yaml`). |
| `--config` | `-c` | Yes* | Path to full config file -- only the `trace_processing` section is used. |

\* Provide either `--trace-processing-config` or `--config`, but not both.

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

Show the current upload and processing status for a model.

```bash
distil model upload-status <model-id>
distil model upload-status <model-id> --output json
```

## Teacher Evaluation

### distil model run-teacher-evaluation

Start a teacher evaluation to validate that a large model can solve your task. This is a feasibility check and performance benchmark before training.

```bash
distil model run-teacher-evaluation <model-id>
```

### distil model teacher-evaluation

Check the status and results of the teacher evaluation.

```bash
distil model teacher-evaluation <model-id>
```

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
