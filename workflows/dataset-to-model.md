# Workflow: Dataset to Trained Model

Train a task-specific small language model from a minimal labeled dataset. This workflow takes you from raw data through teacher evaluation, analysis, iteration, training, and final evaluation.

## Parameters

This workflow adapts based on what the user provides:

- **USE_CASE** — what the model should do (e.g., "redact PII from text", "classify support tickets")
- **TASK_TYPE** — one of the six supported task types (determine using `references/task-selection-guide.md` if the user isn't sure)
- **DATA_LOCATION** — where the user's data lives (directory path, or individual files)
- **MODEL_ID** — created in Step 1, used throughout

---

## Step 0: Preflight

Two quick checks before doing anything else:

**1. Verify Distil CLI authentication.** Follow `references/tasks/verify-auth.md` — run `distil whoami` and, if needed, instruct the user to run `! distil login` in their shell. `/login` in Claude Code is *not* the same as `distil login`.

**2. Update the CLI.** The platform evolves quickly and recent commands may be missing on older versions:

```bash
distil update
```

If you see "Latest available version is X (currently running Y)" in any command output during the workflow — stop, run `distil update`, then re-run the command.

---

## Step 1: Create Model

Deterministic. Run directly in Claude Code.

```bash
distil model create <descriptive-name>
```

Capture the model ID from the output — it's used in every subsequent command.

The run log (`model-building-log-<descriptive-name>.md`) should already have been initialized by the top-level router (`SKILL.md`). Append an entry for the `distil model create` step. See `references/tasks/maintain-run-log.md`.

---

## Step 2: Prepare Data

This step requires judgment. Read these reference files before starting:
- `references/tasks/prepare-data/overview.md` (always)
- The task-specific file from `references/tasks/prepare-data/<task-type>.md`
- `references/task-selection-guide.md` (if task type is unclear)
- `references/model-catalog.md` (for student/teacher selection)
- `references/job-description-guide.md` (for writing `job_description.json`)
- `references/configuration.md` (for choosing the right config beyond student/teacher/task)

### 2a. Understand the user's data

Before writing any files, ask and determine:

1. **What does the input look like?** — Get 2-3 real examples of the text the model will process in production. This shapes the `question` column and `input_description`.
2. **What should the output look like?** — Get 2-3 examples of correct outputs. This shapes the `answer` column and `task_description`.
3. **What makes an answer correct or wrong?** — Specific criteria, not vague quality. This shapes `llm_as_a_judge_instructions`.
4. **What models to use?** — If the user hasn't specified, default to `Llama-3.2-1B-Instruct` (student) and `openai.gpt-oss-120b` (teacher). Only suggest alternatives if there's a concrete reason (tool calling → needs Qwen3/Llama, edge deployment → smaller model, etc.).

### 2b. Write job_description.json

The job description is the most important file — it's the prompt that drives both teacher evaluation and synthetic data generation. Write it with the same care you'd write a production system prompt.

**Structure by task type:**

For `question-answering`:
```json
{
  "task_description": "<What the model should do. Be comprehensive: cover output format, edge cases, what to include and exclude. More useful detail = better synthetic data.>",
  "input_description": "<What the input data looks like. Cover formats, domains, variations, noise patterns.>",
  "llm_as_a_judge_instructions": "<Specific pass/fail criteria. Define what 'correct' means precisely — vague instructions produce noisy evaluation.>"
}
```

For `classification` — add `classes_description`:
```json
{
  "task_description": "...",
  "classes_description": {
    "class_name_1": "Description of when this class applies",
    "class_name_2": "Description of when this class applies"
  },
  "llm_as_a_judge_instructions": "..."
}
```

For `tool-calling-closed-book` or `multi-turn-tool-calling-closed-book` — add `tools`:
```json
{
  "task_description": "...",
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "tool_name",
        "description": "What this tool does",
        "parameters": { "type": "object", "properties": { ... }, "required": [ ... ] }
      }
    }
  ],
  "llm_as_a_judge_instructions": "..."
}
```

**Judgment: what makes a good job description**

A good `task_description` is comprehensive without being verbose. It should:
- Describe the output format with examples (especially for structured output like JSON)
- List what to include AND what to exclude
- Cover edge cases the model will encounter
- Read like a clear specification, not vague guidance

A good `llm_as_a_judge_instructions` defines binary good/bad criteria:
- What specific checks must pass for the answer to be "good"
- What fields or values must match
- What can be ignored (order, whitespace, casing)

### 2c. Write config.yaml

Start minimal. Only include parameters from the "always set" and "set when iterating" tiers (see `references/configuration.md`):

```yaml
base:
  task: <task-type>
  student_model_name: <student-model>
  teacher_model_name: <teacher-model>
```

Add synthgen parameters only when you have a reason:
- `output_is_json: true` — if outputs must be valid JSON
- `generation_target` — if the default 10,000 is too many or too few
- `basic_mutators_to_use` / `mutation_topics` — not required on the first run, but worth setting upfront if the user named specific patterns, scenarios, or length characteristics the data should cover (e.g., "short/medium/long conversations", a list of domains). Otherwise leave defaults and revisit during iteration (Step 5). See `references/mutations-guide.md`.

### 2d. Write train.csv and test.csv

If the user has raw data, transform it into the task-specific format. If the user has no data, help them write examples from scratch.

**Judgment: data quality**

- Aim for 20-50 training examples and 20+ test examples
- Training examples should be diverse and representative — cover different input types, edge cases, difficulty levels
- Test examples should cover the full distribution the model will see in production
- Keep answer formatting consistent across all examples
- For classification: ensure every class appears in the training set

**Deterministic: run the validation checklist**

After writing the files, verify mechanically (read each file and check):
- [ ] train file has ≥ 20 rows
- [ ] test file has ≥ 20 rows
- [ ] All required columns present for the task type
- [ ] No empty values in required columns
- [ ] For classification: every class in `classes_description` appears in train data
- [ ] For tool calling: every `answer` parses as valid JSON
- [ ] `job_description.json` is valid JSON
- [ ] `config.yaml` is valid YAML with `base.task` set

If you write a validation script (Python, jq pipeline, etc.) and it raises an exception, that's a failure — read the traceback, fix the underlying issue, and re-run. Do not declare data "correct" because the script "ran without producing a checklist failure" — an unhandled exception means the checks weren't completed. See `references/tasks/prepare-data/overview.md` for more.

### 2e. Upload data

```bash
distil model upload-data <model-id> --data ./data-dir
```

Check upload status:
```bash
distil model upload-status <model-id>
```

**Log:** append a log entry noting the upload (`references/tasks/maintain-run-log.md`).

---

## Step 3: Run Teacher Evaluation

Deterministic. Run directly.

**Before running, set expectations.** Tell the user the thresholds the report in Step 4 will apply, so they can override if their use case has different requirements:

> *"After this completes, we'll evaluate the teacher against:*
> - *PROCEED ≥ 0.80 (text generation) or ≥ 0.70 (tool calling) — proceed straight to training*
> - *ITERATE 0.60–0.80 (text) or below 0.70 (tool calling) — refine job description and re-evaluate*
> - *RETHINK < 0.60 — task may be under-specified*
>
> *Tell me now if you want different thresholds for your use case."*

Then run:

```bash
distil model run-teacher-evaluation <model-id>
```

This takes a few minutes. Poll until complete using the canonical pattern from `references/tasks/polling-jobs.md` (substitute `teacher-evaluation` as the status command, `sleep 60`). Do not proceed until status is `JOB_SUCCESS`. Do not write your own grep-based loop — the canonical pattern is the only one that reliably catches the actual status values.

---

## Step 4: Analyze Teacher Evaluation

This is where the workflow produces its first major output. The goal is a structured analysis that makes iteration decisions obvious.

### 4a. Gather data

Working directory for this step: the current `iteration-<N>/` (see `workflows/improving-a-model.md`'s Iteration Discipline section). Pass it as the working dir to `references/tasks/analyze-predictions.md`.

1. Get aggregate metrics — **always use `--output json`**, the default text output omits LLM-as-a-Judge and other metrics:
```bash
distil model teacher-evaluation <model-id> --output json | jq '.aggregateMetrics'
```

2. Download per-example teacher predictions into the iteration dir (see `references/tasks/retrieve-predictions.md` for full options):
```bash
distil model download-teacher-evaluation-predictions <model-id> \
  --file-name iteration-<N>/teacher-predictions.jsonl
```

Then load into a dataframe for analysis.

3. Read the uploaded data files (job_description.json, config.yaml, test set) for the report context.

### 4b. Produce the Teacher Evaluation Analysis Report

Use the **Teacher Evaluation Analysis Report** template from `references/tasks/analyze-predictions.md`. The report is the basis for all iteration decisions and produces a verdict (PROCEED / ITERATE / RETHINK).

Save the report as `iteration-<N>/teacher-eval-analysis.md`. **After writing the report, tell the user**: the file path, the headline metric, and the verdict. Don't bury the report — it's the main output of this step and the user needs to see it to decide on next steps.

**Log:** append a log entry with the verdict and headline (`references/tasks/maintain-run-log.md`).

### 4c. Decision point

Based on the report:

- **PROCEED** → Go to Step 6 (Confirm Before Training).
- **ITERATE** → Switch to `workflows/improving-a-model.md` → **Entry Point A** (ITERATE path — work through job description, data, synthgen/mutations, teacher model in that order). The report's "Recommended actions" section tells you which lever to start with.
- **RETHINK** → Switch to `workflows/improving-a-model.md` → **Entry Point A** (RETHINK path — step back and question task type, task definition, judge instructions before touching individual levers).

**Also switch to Entry Point A if the teacher eval scores look questionable for any reason** — e.g. the judge flagged outputs as "bad" that look correct on inspection, or the metrics look inconsistent. The RETHINK path is the right place to confirm the judge is telling the truth before iterating on the model.

---

## Step 5: Iterate (return point)

This is the return point from `workflows/improving-a-model.md`. After each iteration loop completes there, come back to Step 4 to analyze the new teacher evaluation results. Repeat until the verdict is `PROCEED`.

---

## Step 6: Confirm Before Training

Training is a **6+ hour, credit-burning operation**. Before kicking it off, confirm with the user:

1. Show the final `config.yaml` contents (or summarize the key fields if it's long).
2. List the student and teacher models being used.
3. Mention the expected duration (~6 hours, but can be longer for larger students).
4. Ask explicitly: *"Reply 'go' (or similar) to start training, or tell me what to change first."*

Do NOT run `distil model run-training` until the user confirms. This checkpoint exists because:
- Training mistakes (wrong model, wrong config) are expensive to discover after 6 hours.
- Once started, training consumes credits that are hard to refund.
- The user often has context the analysis report doesn't (deployment constraints, deadlines, budget).

---

## Step 7: Train

Deterministic. Run only after the user confirms in Step 6.

```bash
distil model run-training <model-id>
```

Training takes several hours (typically 6+). The pipeline has three stages:
1. **Evaluate Teacher** — confirms teacher performance on the test set
2. **Generate Synthetic Data** — teacher generates training data from your examples
3. **Finetune Student** — student model is trained on the synthetic data, then evaluated

Poll until complete using the canonical pattern from `references/tasks/polling-jobs.md`. Swap `sleep 60` for `sleep 600` — training is multi-hour, so minute-scale polling is wasteful.

Suggest the user check back periodically. They can close the session and come back — `distil model training <model-id>` always shows current state.

Do not proceed to Step 8 until status is `JOB_SUCCESS`.

---

## Step 8: Analyze Training Results

Same analysis pattern as Step 4, but now comparing the trained student model against the teacher and base student baselines.

### 8a. Gather data

Working directory for this step: the current `iteration-<N>/` (see `workflows/improving-a-model.md`'s Iteration Discipline section). Pass it as the working dir to `references/tasks/analyze-predictions.md`.

1. Get aggregate metrics — **always use `--output json`**, the default text output omits LLM-as-a-Judge and other metrics:
```bash
distil model training <model-id> --output json | jq '.aggregateMetrics'
```

2. Download tuned (finetuned) student predictions into the iteration dir:
```bash
distil model download-training-predictions <model-id> \
  --file-name iteration-<N>/student-predictions.jsonl
```

3. Download base student predictions (only available via API — see `references/tasks/retrieve-predictions.md`):
```python
response = requests.get(
    f"https://api.distillabs.ai/trainings/{training_job_id}/evaluation-results",
    headers=auth_header,
)
base_predictions_url = response.json()["base_student_evaluation_predictions_download_url"]
```

4. Also load the teacher predictions saved in Step 4 (`iteration-<N>/teacher-predictions.jsonl`). You now have three sets of predictions to compare: **base student** (untuned baseline), **teacher** (upper bound), and **tuned student** (the trained model).

### 8b. Produce the Training Analysis Report

Use the **Training Analysis Report** template from `references/tasks/analyze-predictions.md`. This is the three-way variant (Base Student / Teacher / Tuned Student) that produces a verdict (DEPLOY / RETUNE / ESCALATE).

Save as `iteration-<N>/training-analysis.md`. **After writing the report, tell the user**: the file path, the headline metric (tuned student primary score), the deltas vs. teacher and base, and the verdict. The user needs this to decide whether to deploy or retune.

**Log:** append a log entry with the verdict and headline (`references/tasks/maintain-run-log.md`).

### 8c. Decision point

- **DEPLOY** → Go to Step 9.
- **RETUNE** → Switch to `workflows/improving-a-model.md` → **Entry Point B** (retune levers: different student, different tuning parameters). After retuning completes, return here and re-run Step 8 analysis.
- **ESCALATE** → Switch to `workflows/improving-a-model.md` → **Entry Point B** (ESCALATE section). Likely needs to go back to Entry Point A — the data or task definition has a fundamental issue.

---

## Step 9: Deploy

Once satisfied with training results, deploy the model.

**Log:** append a final entry noting deployment (`references/tasks/maintain-run-log.md`).

### Local deployment (recommended for testing)

```bash
distil model download <model-id>
distil model deploy local <model-id>
distil model invoke <model-id>  # Get the invocation command
```

### Remote deployment (testing only, not production)

```bash
distil model deploy remote <model-id>
distil model invoke <model-id>
```

See `references/tasks/deployment-integration.md` for details on deployment options, the OpenAI-compatible API, and production deployment.

---

## Reference Files

This workflow draws on these reference files. Read them when you need details on a specific step:

| Step | Reference |
|------|-----------|
| Task selection | `references/task-selection-guide.md` |
| Model catalog and compatibility | `references/model-catalog.md` |
| Job description authoring | `references/job-description-guide.md` |
| Data format | `references/tasks/prepare-data/overview.md` + task-specific file |
| Configuration | `references/configuration.md` |
| Upload | `references/tasks/upload-dataset.md` |
| Teacher evaluation | `references/tasks/teacher-evaluation.md` |
| Metrics interpretation | `references/evaluation-metrics.md` |
| Predictions download | `references/tasks/retrieve-predictions.md` |
| Analysis report templates | `references/tasks/analyze-predictions.md` |
| Polling long-running jobs | `references/tasks/polling-jobs.md` |
| Iteration (teacher eval or training) + `iteration-N/` convention | `workflows/improving-a-model.md` |
| Run log format and triggers | `references/tasks/maintain-run-log.md` |
| API authentication | `references/api-reference.md` |
| Training | `references/tasks/training.md` |
| Deployment | `references/tasks/deployment-integration.md` |
| Mutations (iteration) | `references/mutations-guide.md` |
