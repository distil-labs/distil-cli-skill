# Workflow: Dataset to Trained Model

Train a task-specific small language model from a minimal labeled dataset. This workflow takes you from raw data through teacher evaluation, analysis, iteration, training, and final evaluation.

## Parameters

This workflow adapts based on what the user provides:

- **USE_CASE** — what the model should do (e.g., "redact PII from text", "classify support tickets")
- **TASK_TYPE** — one of the six supported task types (determine using `references/task-selection-guide.md` if the user isn't sure)
- **DATA_LOCATION** — where the user's data lives (directory path, or individual files)
- **Entity IDs** — produced as you go, not up front. `<upload-id>` (Step 2f) → `<teacher-evaluation-id>` (Step 3) and `<training-dataset-id>` (Step 7) → `<slm-id>` (Step 8) → `<deployment-id>` (Step 10). Read each out of its command's output. They are also linked on the platform, so a lost one is recoverable — see `references/cli-reference.md` (`### Tracing the Chain`).

---

## Step 0: Preflight

Two quick checks before doing anything else:

**1. Verify Distil CLI authentication.** Follow `references/tasks/verify-auth.md` — run `distil whoami` and, if needed, instruct the user to run `! distil auth` in their shell. `/login` in Claude Code is *not* the same as `distil auth`.

**2. Update the CLI.** The platform evolves quickly and recent commands may be missing on older versions:

```bash
distil update
```

If you see "Latest available version is X (currently running Y)" in any command output during the workflow — stop, run `distil update`, then re-run the command.

---

## Step 1: Set Up the Run Log

There is nothing to create on the platform up front — every entity is created by the step that produces it. So this step is bookkeeping only.

The run log (`model-building-log-<descriptive-name>.md`) should already have been initialized by the top-level router (`SKILL.md`). Confirm it exists. It captures decisions and reasoning as you go — see `references/tasks/maintain-run-log.md` for the entry format and append triggers.

Each of the steps below produces one entity, and the ID it prints is the input to the next:

```
<upload-id> → <teacher-evaluation-id>                              (Steps 2f, 3)
<upload-id> → <training-dataset-id> → <slm-id> → <deployment-id>   (Steps 7, 8, 10)
```

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

1. **What does the input look like?** — Get 2-3 real examples of the text the model will process in production. This shapes the `user` turns (and `input_description`, if the task is `question-answering` — that's the only task type where synthgen reads it).
2. **What should the output look like?** — Get 2-3 examples of correct outputs. This shapes the `assistant` turns and `task_description`.
3. **What makes an answer correct or wrong?** — Specific criteria, not vague quality. This shapes `llm_as_a_judge_instructions`.
4. **What models to use?** — If the user hasn't specified, default to `Llama-3.2-1B-Instruct` (student) and `openai.gpt-oss-120b` (teacher). Only suggest alternatives if there's a concrete reason (tool calling → needs Qwen3/Llama, edge deployment → smaller model, etc.).

### 2b. Write job_description.json

The job description is the most important file — it's the prompt that drives both teacher evaluation and synthetic data generation. Write it with the same care you'd write a production system prompt.

**Structure by task type:**

For `question-answering`:
```json
{
  "task_description": "<What the model should do. Be comprehensive: cover output format, edge cases, what to include and exclude.>",
  "input_description": "<What the input data looks like — formats, domains, variations, noise patterns, with at least one concrete example. Required for QA, and must be self-contained: synthgen reads this (not task_description) when generating new inputs.>",
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

### 2d. Write train.jsonl and test.jsonl

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
- [ ] Every row has a `messages` array with a `user` turn and an `assistant` turn (plus a `context` field for open-book QA)
- [ ] No empty values in required fields
- [ ] For classification: every class in `classes_description` appears in train data
- [ ] For tool calling: every assistant `tool_calls[].function.arguments` is a valid JSON object (HuggingFace format, not a stringified blob)
- [ ] Over-length examples surfaced and confirmed with the user: the platform silently truncates data above its length limits (structured and unstructured differ, see `references/configuration.md`), so flag the longest examples and confirm before uploading
- [ ] No train/test leakage: no test row duplicates or near-duplicates a train row (near-duplicate: differs only in whitespace, casing, or trivial punctuation)
- [ ] `job_description.json` is valid JSON
- [ ] `config.yaml` is valid YAML with `base.task` set

If you write a validation script (Python, jq pipeline, etc.) and it raises an exception, that's a failure — read the traceback, fix the underlying issue, and re-run. Do not declare data "correct" because the script "ran without producing a checklist failure" — an unhandled exception means the checks weren't completed. See `references/tasks/prepare-data/overview.md` for more.

### 2e. Data consistency analysis

Before uploading, run the quantitative analysis from `references/tasks/analyze-uploads.md` against the local train/test files (the data is local, so no download is needed). That file's Quantitative Section is the authoritative check list; reuse the length and leakage numbers you already computed for the Step 2d checklist rather than recomputing them.

This is a pre-upload run: no report file is needed. Summarize the findings for the user in 3-5 lines. If anything is flagged, fix it before uploading; teacher evaluation costs credits, and a distribution problem found now is one iteration saved later. In particular, if any example is long enough to risk truncation, confirm with the user before uploading rather than letting the platform cut it silently. Then offer the qualitative deep dive (categorical axes + job-description cross-check) from the same file; if the user declines, continue.

### 2f. Upload data

```bash
distil upload create --data ./data-dir
# Output: Upload successful. Upload ID: <upload-id>
```

Capture the `<upload-id>` — Steps 3 and 6 both take it. Check upload status:

```bash
distil upload status <upload-id> --output json | jq -r '.status'
```

An upload from local files is complete as soon as it exists, so this should report `JOB_SUCCESS` immediately.

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
distil teacher-evaluation create-from-upload <upload-id>
# Output: Teacher evaluation started. Teacher Evaluation ID: <teacher-evaluation-id>
```

Capture the `<teacher-evaluation-id>` from the output — the analysis in Step 4 needs it.

This takes a few minutes. Poll until complete using the canonical pattern from `references/tasks/polling-jobs.md` (`distil teacher-evaluation status <teacher-evaluation-id> --output json`, `sleep 60`). Do not proceed until status is `JOB_SUCCESS`. Do not write your own grep-based loop — the canonical pattern is the only one that reliably catches the actual status values.

---

## Step 4: Analyze Teacher Evaluation

This is where the workflow produces its first major output. The goal is a structured analysis that makes iteration decisions obvious.

### 4a. Gather data

Working directory for this step: the current `iteration-<N>/` (see `workflows/improving-a-model.md`'s Iteration Discipline section). Pass it as the working dir to `references/tasks/analyze-predictions.md`.

1. Get aggregate metrics — **always use `--output json`**, the default text output omits LLM-as-a-Judge and other metrics:
```bash
distil teacher-evaluation metrics <teacher-evaluation-id> --output json | jq '.teacher_performance'
```

2. Download per-example teacher predictions into the iteration dir (see `references/tasks/retrieve-predictions.md` for full options):
```bash
distil teacher-evaluation download-predictions <teacher-evaluation-id> \
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

Getting from an upload to a trained SLM is two commands, and **both are billable**: synthetic data generation costs 2 credits, and training is a 6+ hour credit-burning run. Confirm once, for the pair, before starting either:

1. Show the final `config.yaml` contents (or summarize the key fields if it's long).
2. List the student and teacher models being used.
3. Mention the expected duration (generation: minutes. training: ~6 hours, longer for larger students).
4. Ask explicitly: *"Reply 'go' (or similar) to start synthetic data generation and then training, or tell me what to change first."*

Do NOT run `distil training-dataset create-from-upload` or `distil slm create-from-training-dataset` until the user confirms. This checkpoint exists because:
- Training mistakes (wrong model, wrong config) are expensive to discover after 6 hours.
- Once started, training consumes credits that are hard to refund.
- The user often has context the analysis report doesn't (deployment constraints, deadlines, budget).

---

## Step 7: Generate the Training Dataset

Deterministic. Run only after the user confirms in Step 6.

```bash
distil training-dataset create-from-upload <upload-id>
# Output: Synthetic data generation started. Training Dataset ID: <training-dataset-id>
```

Costs 2 credits and takes minutes, not hours. Capture the `<training-dataset-id>` from the output. Poll until `JOB_SUCCESS` per `references/tasks/polling-jobs.md` (`sleep 60`).

### 7a. Look at what was generated

This is free, returns in seconds, and is the cheapest quality gate in the whole workflow. Do it every time:

```bash
distil training-dataset sample <training-dataset-id>
```

If the sampled rows look wrong — off-topic, malformed, wrong label distribution, truncated content — that is a synthgen or job-description problem, and training on them wastes six hours. Go to `workflows/improving-a-model.md` → **Entry Point A** instead of continuing to Step 8.

**Log:** append an entry noting the dataset and what the sample showed (`references/tasks/maintain-run-log.md`).

---

## Step 8: Train

Deterministic. Run once the sample in Step 7a looks right.

```bash
distil slm create-from-training-dataset <training-dataset-id>
# Output: Training started. SLM ID: <slm-id>
```

Capture the `<slm-id>` from the output.

Training takes several hours (typically 6+): the student model is fine-tuned on the generated data, then evaluated against your test set.

Poll until complete using the canonical pattern from `references/tasks/polling-jobs.md` (`distil slm status <slm-id> --output json`). Swap `sleep 60` for `sleep 600` — training is multi-hour, so minute-scale polling is wasteful.

Suggest the user check back periodically. They can close the session and come back — `distil slm status <slm-id>` always shows current state, and `distil slm logs <slm-id>` explains a failure.

Do not proceed to Step 9 until status is `JOB_SUCCESS`.

---

## Step 9: Analyze Training Results

Same analysis pattern as Step 4, but now comparing the trained student model against the teacher and base student baselines.

### 9a. Gather data

Working directory for this step: the current `iteration-<N>/` (see `workflows/improving-a-model.md`'s Iteration Discipline section). Pass it as the working dir to `references/tasks/analyze-predictions.md`.

1. Get aggregate metrics — **always use `--output json`**, the default text output omits LLM-as-a-Judge and other metrics. `distil slm metrics` reports the tuned student and the untuned baseline side by side:
```bash
distil slm metrics <slm-id> --output json | jq '.tuned_model_performance'
distil slm metrics <slm-id> --output json | jq '.base_model_performance'
```

2. Download tuned (finetuned) student predictions into the iteration dir:
```bash
distil slm download-predictions <slm-id> \
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

### 9b. Produce the Training Analysis Report

Use the **Training Analysis Report** template from `references/tasks/analyze-predictions.md`. This is the three-way variant (Base Student / Teacher / Tuned Student) that produces a verdict (DEPLOY / RETUNE / ESCALATE).

Save as `iteration-<N>/training-analysis.md`. **After writing the report, tell the user**: the file path, the headline metric (tuned student primary score), the deltas vs. teacher and base, and the verdict. The user needs this to decide whether to deploy or try different tuning.

**Log:** append a log entry with the verdict and headline (`references/tasks/maintain-run-log.md`).

### 9c. Decision point

- **DEPLOY** → Go to Step 10.
- **RETUNE** → Switch to `workflows/improving-a-model.md` → **Entry Point B** (different student, different tuning parameters). After the new SLM finishes, return here and re-run Step 9 analysis.
- **ESCALATE** → Switch to `workflows/improving-a-model.md` → **Entry Point B** (ESCALATE section). Likely needs to go back to Entry Point A — the data or task definition has a fundamental issue.

---

## Step 10: Deploy

Once satisfied with training results, serve the model.

**Log:** append a final entry noting the deployment and its ID (`references/tasks/maintain-run-log.md`).

### Hosted deployment (recommended for testing)

```bash
distil deployment create-from-slm <slm-id>
# Output: Deployment started. Deployment ID: <deployment-id>

distil deployment status <deployment-id>     # wait for endpoint_status == "running"
distil deployment endpoint <deployment-id>   # prints the URL and API key
```

Poll on `endpoint_status`, not `deployment_status` — a finished deploy whose endpoint is still `stopped` cannot answer a request yet. Remind the user to shut it down when they are done testing, since a running deployment consumes inference credits:

```bash
distil deployment delete <deployment-id>
```

### Local serving

```bash
distil slm download <slm-id> --destination ./my-slm   # model.tar + config.yaml
```

Extract the tarball and serve `model/` with vLLM. See `references/tasks/deployment-integration.md` for the per-backend commands (vLLM, llama-cpp, Ollama), the OpenAI-compatible API, and production deployment.

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
| Data consistency analysis (Step 2e) | `references/tasks/analyze-uploads.md` |
| Upload | `references/tasks/upload-dataset.md` |
| Teacher evaluation | `references/tasks/teacher-evaluation.md` |
| Training datasets and `sample` (Step 7) | `references/cli-reference.md` (`## Training Datasets`) |
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
