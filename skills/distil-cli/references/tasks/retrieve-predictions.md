# Retrieve Predictions

Download per-example predictions from trace processing, teacher evaluation, and model training. These predictions let you inspect what each model produced for each test example, enabling detailed error analysis and debugging.

**This is the canonical guide.** Workflows and other tasks should reference this file rather than re-documenting the commands.

**Prerequisites:** For API-based downloads, see `references/api-reference.md` for the `distil_bearer_token()` implementation and auth header setup. CLI commands work as long as you're logged in (`distil auth`).

---

## Trace Processing Predictions

Once trace processing completes, download the original production model's predictions on the test set — the model that generated the traces. Use these to compare the original model against the committee-relabeled ground truth.

### CLI

```bash
distil upload download-traces-predictions <upload-id>

# Custom output filename
distil upload download-traces-predictions <upload-id> --file-name predictions.jsonl
```

Default output: `<upload-id>-traces-predictions.jsonl`

### API (alternative)

The download URL is included in the upload metrics response once processing completes:

```python
response = requests.get(
    f"https://api.distillabs.ai/uploads/{upload_id}/metrics",
    headers=auth_header,
)
download_url = response.json()["base_model_predictions_download_url"]
```

```bash
curl -o base_model_predictions.jsonl "<DOWNLOAD_URL>"
```

---

## Teacher Evaluation Predictions

After teacher evaluation completes, download the teacher model's predictions on your test set. Use these to understand where the teacher succeeds or fails before committing to training.

### CLI

```bash
distil teacher-evaluation download-predictions <teacher-evaluation-id>

# Custom output filename
distil teacher-evaluation download-predictions <teacher-evaluation-id> --file-name teacher-predictions.jsonl
```

Default output: `<teacher-evaluation-id>-teacher-evaluation-predictions.jsonl`. Reads the URL from `distil teacher-evaluation metrics <teacher-evaluation-id>`, so the job has to have succeeded first.

### API (alternative)

The download URL is included in the evaluation metrics response once the job completes:

```python
response = requests.get(
    f"https://api.distillabs.ai/teacher-evaluations/{eval_job_id}/metrics",
    headers=auth_header,
)
download_url = response.json()["predictions_download_url"]
```

```bash
curl -o teacher_evaluation_predictions.jsonl "<DOWNLOAD_URL>"
```

---

## Training Predictions

After model training completes, download the tuned (fine-tuned) student model's predictions on the test set. Compare these against the teacher predictions to assess how well the student learned, and against the base student to see what training improved.

### CLI

```bash
distil slm download-predictions <slm-id>

# Custom output filename
distil slm download-predictions <slm-id> --file-name student-predictions.jsonl
```

Default output: `<slm-id>-slm-predictions.jsonl`. Reads the URL from `distil slm metrics <slm-id>`, so the training job has to have succeeded first.

For the **base student** (untuned) predictions, there is no CLI command — see the API section below. `distil slm metrics <slm-id>` does report the base student's aggregate scores at `.base_model_performance`, so you can compare summary numbers without the per-example file.

### API (alternative)

For the **tuned student** predictions, the download URL is in the evaluation results response:

```python
response = requests.get(
    f"https://api.distillabs.ai/trainings/{training_job_id}/evaluation-results",
    headers=auth_header,
)
results = response.json()
tuned_url = results["finetuned_student_evaluation_predictions_download_url"]
```

For the **base student** (untuned) predictions — only available via API:

```python
base_url = results["base_student_evaluation_predictions_download_url"]
```

The base student predictions are useful for the training analysis report — they show the baseline before fine-tuning, so you can quantify what training added (and check for regressions).

---

## Reading Predictions

All prediction files are in JSON Lines format. Load them with pandas:

```python
import pandas as pd

df = pd.read_json("predictions.jsonl", lines=True)
print(df.head())
```

## When to Use Predictions

- **After trace processing** → compare the original production model against the committee-relabeled ground truth (see `references/tasks/analyze-predictions.md` "Original Model Analysis Report").
- **After teacher evaluation** → if scores are borderline, inspect predictions to decide whether to iterate or proceed (see `references/tasks/analyze-predictions.md` "Teacher Evaluation Analysis Report").
- **After training** → compare tuned student vs. teacher and base student to find where the student diverges. Drives iteration decisions: refine job description, add examples for failure cases, or try a larger student model (see `references/tasks/analyze-predictions.md` "Training Analysis Report").
