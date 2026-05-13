# Upload Dataset

Upload your prepared data files to the Distil Labs platform for teacher evaluation and training.

## Upload Data

The recommended approach is to place all required files in a single directory and upload the directory:

```bash
distil model upload-data <model-id> --data <directory>
```

The directory should contain files with these standard names:

| File | Required | Description |
|------|----------|-------------|
| `job_description.json` | Yes | Task objectives and configuration |
| `train.csv` or `train.jsonl` | Yes | 20+ labeled (question, answer) pairs |
| `test.csv` or `test.jsonl` | Yes | Held-out evaluation set |
| `config.yaml` | Yes | Task type, student model, teacher model, and training parameters |
| `unstructured.csv` | No | Domain text for synthetic data generation |

### Individual File Flags

As an alternative to directory mode, you can specify each file individually:

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

## What Happens After Upload

After upload, the platform:

1. Validates the files for correct format and required fields.
2. Checks that the data matches the task type declared in `config.yaml`.
3. Prepares the data for teacher evaluation.

Once validation completes, you can proceed to teacher evaluation with `distil model run-teacher-evaluation <model-id>`.

## Checking Upload Status

Check whether the upload has been validated and is ready:

```bash
distil model upload-status <model-id>
```

For machine-readable output:

```bash
distil model upload-status <model-id> --output json
```

## Downloading Uploaded Data

To verify what was uploaded, download the data files back to your machine:

```bash
distil model download-data <model-id>
```

This is useful for confirming the correct files were sent, especially when debugging issues with teacher evaluation or training.

## Common Issues

### Wrong file format
The platform expects CSV (`.csv`) or JSONL (`.jsonl`) for train and test files, JSON (`.json`) for job descriptions, and YAML (`.yaml`) or JSON (`.json`) for config. Uploading files in other formats will cause validation errors.

### Missing required files
When using directory mode, the directory must contain `job_description.json` and training/test data files. When using individual flags, you must provide at least `--job-description`, `--train`, and `--test`.

### Too few examples
Training requires a minimum of 20 examples in the training set. If you have fewer, the platform will reject the upload. Add more labeled examples before uploading.

### Data does not match task type
The columns and fields in your data files must match what the selected task type expects. For example, classification tasks require a `question` and `answer` column, while open book QA tasks also require a `context` column. See the data preparation guide for your specific task type.

### Inconsistent labels
For classification tasks, make sure the labels in your training data match the classes described in `job_description.json`. Mismatches between the data and the job description will lead to poor teacher evaluation results.
