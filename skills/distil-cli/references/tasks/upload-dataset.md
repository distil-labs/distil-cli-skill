# Upload Dataset

Upload your prepared data files to the Distil Labs platform for teacher evaluation and training.

## Upload Data

The recommended approach is to place all required files in a single directory and upload the directory:

```bash
distil upload create --data <directory>
# Output: Upload successful. Upload ID: <upload-id>
```

Capture the `<upload-id>` — it is what teacher evaluation and synthetic data generation both consume.

The directory should contain files with these standard names:

| File | Required | Description |
|------|----------|-------------|
| `job_description.json` | Yes | Task objectives and configuration |
| `train.jsonl` | Yes | 20+ labeled examples, each a `messages` conversation |
| `test.jsonl` | Yes | Held-out evaluation set |
| `config.yaml` | Yes | Task type, student model, teacher model, and training parameters |
| `unstructured.jsonl` | No | Domain text for synthetic data generation |

### Individual File Flags

As an alternative to directory mode, you can specify each file individually:

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

`--config` is required either way. In directory mode the directory must hold `config.yaml` or `config.yml`. The command fails before uploading anything if it is missing, so a missing config is a fast failure rather than a half-done upload.

## What Happens After Upload

After upload, the platform:

1. Validates the files for correct format and required fields.
2. Checks that the data matches the task type declared in `config.yaml`.
3. Prepares the data for teacher evaluation.

Once validation completes, you can proceed to teacher evaluation with `distil teacher-evaluation create-from-upload <upload-id>`.

## Checking Upload Status

Check whether the upload has been validated and is ready:

```bash
distil upload status <upload-id>
distil upload status <upload-id> --output json | jq -r '.status'
```

Uploads created from local files are complete the moment they exist and have no logs. Uploads built from prepared traces process asynchronously, so poll until `JOB_SUCCESS` and read `distil upload logs <upload-id>` if one fails.

`distil upload list` shows every upload you have, newest first — useful for recovering an ID you lost:

```bash
distil upload list --output json | jq -r '.[0].id // "none"'
```

`distil upload metrics <upload-id>` reports `base_model_performance` for the untuned student, when there is any.

## Downloading Uploaded Data

To verify what was uploaded, download the data files back to your machine:

```bash
distil upload download <upload-id> --destination <directory>
```

This writes `train.jsonl`, `test.jsonl`, `unstructured.jsonl`, `config.yaml`, and `job_description.json` — the same filenames `--data` expects, so a download can be re-uploaded without renaming anything.

This is useful for confirming the correct files were sent, especially when debugging issues with teacher evaluation or training.

## Common Issues

### Wrong file format
The platform expects JSONL (`.jsonl`) for train and test files, JSON (`.json`) for job descriptions, and YAML (`.yaml`/`.yml`) for config — `config.json` is no longer accepted. Uploading files in other formats will cause validation errors.

### Missing required files
When using directory mode, the directory must contain `job_description.json`, training/test data files, and `config.yaml` (or `config.yml`). When using individual flags, you must provide `--job-description`, `--train`, `--test`, and `--config`. A missing config fails immediately, before anything is uploaded.

### Too few examples
Training requires a minimum of 20 examples in the training set. If you have fewer, the platform will reject the upload. Add more labeled examples before uploading.

### Data does not match task type
The fields in your data files must match what the selected task type expects. Every task type requires a `messages` conversation per example; open book QA additionally requires a sibling `context` field. See the data preparation guide for your specific task type.

### Inconsistent labels
For classification tasks, make sure the labels in your training data match the classes described in `job_description.json`. Mismatches between the data and the job description will lead to poor teacher evaluation results.
