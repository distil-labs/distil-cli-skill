# Workflow: Traces to Trained Model

Train a task-specific small language model from production LLM traces. This workflow takes you from raw production logs through trace processing, baseline analysis, teacher evaluation, iteration, training, and deployment.

This is the right workflow when the user already has an LLM-powered application running in production and wants to distill its behavior into a smaller, cheaper model.

## Parameters

This workflow adapts based on what the user provides:

- **USE_CASE** — what the model should do (e.g., "extract structured data from support tickets", "classify customer intent")
- **TASK_TYPE** — one of the supported task types. **Note:** `question-answering-open-book` is not supported via traces. If the user has RAG-style traces with context embedded in the user message, use `question-answering` and put the full prompt in the `question` field (see `references/tasks/upload-and-process-traces.md`).
- **TRACES_LOCATION** — where the raw production logs live (e.g. Langfuse export, OpenAI logs)
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

---

## Step 2: Prepare Traces

This step requires judgment. Read these reference files before starting:
- `references/tasks/upload-and-process-traces.md` (always)
- `references/task-selection-guide.md` (if task type is unclear)
- `references/model-catalog.md` (for student/teacher selection)
- `references/job-description-guide.md` (for writing `job_description.json`)
- `references/configuration.md` (for choosing the right config)

### 2a. Analyze raw traces

Before writing any conversion code, inspect the raw traces to understand what you're working with:

1. **Input structure** — What are the template variables? What's constant across traces vs. variable per request? What's the message format (system + user, or just user)?
2. **Output structure** — Plain text, JSON, structured fields? If JSON, what's the schema?
3. **Original model** — Which production model generated these traces? (You'll compare against it later.)
4. **Token counts** — Typical input/output sizes. Long inputs may hit `validation_max_total_length` (10,000 char default).
5. **Label distribution** — For classification: count per class. For QA: answer length distribution. Look for class imbalance.

Share the findings with the user to make sure they agree with the assessment. Only proceed when they confirm. You'll reuse this when writing `job_description.json` and the analysis reports.

### 2b. Write the conversion script

Convert raw traces into the `openai_messages` format expected by the platform. Each line of `traces.jsonl` looks like:

```json
{"messages": [
  {"role": "system", "content": "<constant system prompt>"},
  {"role": "user", "content": "<variable user input>"},
  {"role": "assistant", "content": "<model output, possibly JSON-stringified>"}
]}
```

**Key decisions when writing the script:**

- **Constant parts of the production prompt** → `system` message
- **Variable parts** (per-request input) → `user` message
- **Model output** → `assistant` message
- **Chain-of-thought reasoning** — if the original model produced reasoning (e.g., `reasoning_content`), include it as an `analysis` field in the assistant's output JSON so the student learns to reason similarly

**Critical gotchas (will silently break uploads):**

- Use `ensure_ascii=True` in Python (or equivalent in other languages). The platform's parser truncates records containing `U+2028` or `U+2029`. See `references/tasks/upload-and-process-traces.md` for details.
- Strip control characters (`\x0b`, etc.) from content before serialization.

### 2c. Write job_description.json

The job description is the most important file — it shapes both teacher evaluation and synthetic data generation. Write it with the same care as a production system prompt.

```json
{
  "task_description": "<What the model should do. Be comprehensive: cover output format, edge cases, what to include and exclude.>",
  "input_description": "<MUST be self-contained. The synthgen model does NOT see task_description — only input_description. Describe input structure, markers, sections, and give examples.>",
  "llm_as_a_judge_instructions": "<Specific pass/fail criteria. Be strict on deterministic fields (labels, scores), lenient on free-text fields (analysis, reasoning).>"
}
```

For task-specific fields (`classes_description` for classification, `tools` for tool calling), see the structure in `workflows/dataset-to-model.md` Step 2b — same format, different task type.

**Judgment: what makes a good job description.** See `references/job-description-guide.md` for the full criteria. The load-bearing point for traces is that `input_description` must be self-contained: synthgen sees it without `task_description`, so if you only describe input structure in `task_description`, synthgen will produce poor data even when teacher evaluation looks fine.

### 2d. Write config.yaml

Start minimal. Include the trace processing section since this workflow uses traces:

```yaml
base:
  task: <task-type>
  student_model_name: <student-model>
  teacher_model_name: <teacher-model>

trace_processing:
  observation_format: openai_messages
  remove_system_prompt_from_traces: true   # if system prompt is large
  compress_job_description: true            # if task_description is long
  # convert_to_single_turn: false          # REQUIRED when task is multi-turn-tool-calling-closed-book
                                            # (default true works for all single-turn tasks)
```

Add synthgen parameters only when you have a reason:
- `output_is_json: true` — if outputs must be valid JSON
- `parallel_llm_calls: true` — speed up synthetic data generation
- `validation_max_total_length: 30000` — if your traces have long inputs (default 10,000 may be too tight)
- `basic_mutators_to_use` / `mutation_topics` — not required on the first run, but worth setting upfront if the user named specific patterns, scenarios, or length characteristics the data should cover (e.g., "short/medium/long conversations"). Otherwise leave defaults and revisit during iteration (Step 7). See `references/mutations-guide.md`.
- `generation_target` — revisit during iteration (Step 7) unless the user has an explicit reason to deviate from the default.

**Default model recommendation:** If the user hasn't specified, default to `Llama-3.2-1B-Instruct` (student) and `openai.gpt-oss-120b` (teacher). For complex production tasks, consider larger students — see `references/model-catalog.md` for the catalog and common pairings.

### 2e. Pre-upload validation

Verify mechanically before uploading:
- [ ] `traces.jsonl` exists and every line is valid JSON
- [ ] `messages` array present in every line
- [ ] No control characters or Unicode line separators (`U+2028`, `U+2029`) in content
- [ ] `job_description.json` is valid JSON
- [ ] `config.yaml` is valid YAML with `base.task` set
- [ ] Task type is NOT `question-answering-open-book` (not supported via traces)

If you write a validation script (Python, jq pipeline, etc.) and it raises an exception, that's a failure — read the traceback, fix the underlying issue, and re-run. Do not declare data "correct" because the script "ran without producing a checklist failure" — an unhandled exception means the checks weren't completed. See `references/tasks/prepare-data/overview.md` for more.

---

## Step 3: Upload and Process Traces

Deterministic. Run directly.

```bash
distil model upload-traces <model-id> --data ./traces-dir
```

Trace processing runs a multi-step pipeline:
1. **Relevance filtering** — removes low-quality or off-task traces
2. **Committee relabeling** — multiple teacher models generate candidate outputs; an arbiter picks the best
3. **Train/test split** — produces `train`, `test`, and `unstructured` splits

Poll until complete using the canonical pattern from `references/tasks/polling-jobs.md` (substitute `upload-status` as the status command, `sleep 60`). Trace processing typically takes several minutes. Do not proceed until status is `JOB_SUCCESS`.

---

## Step 4: Analyze Original Model Performance

Trace processing produces a "ground truth" via committee relabeling. This step compares the original production model (whose outputs are in the traces) against that ground truth — telling you how much relabeling changed the data.

This step doesn't gate the workflow; it gives context for everything that follows.

### 4a. Gather data

Download per-example predictions of the original model (see `references/tasks/retrieve-predictions.md` for full options):

```bash
distil model download-traces-predictions <model-id> --file-name original-model-predictions.jsonl
```

Read the resulting JSONL into a dataframe.

### 4b. Produce the Original Model Analysis Report

Use the **Original Model Analysis Report** template from `references/tasks/analyze-predictions.md`. This is the single-model variant with INFORMATIONAL verdict (no gate).

Save the report (e.g., `original-model-analysis.md`). **After writing the report, tell the user**: the file path, the headline agreement metric, and the takeaway (high/moderate/low agreement). The user needs to see this to understand the baseline before continuing.

### 4c. Decision point and data fixes

Based on the report:

- **High agreement (≥ 0.8)** → relabeling didn't change much; the production data is already high quality. Continue to Step 5.
- **Moderate agreement (0.5–0.8)** → relabeling improved data quality. This is normal for noisy production traces. Continue to Step 5.
- **Low agreement (< 0.5)** → relabeling significantly changed the data. Inspect the disagreement samples to confirm the relabeling is correct (not introducing noise). If correct, continue.
- **Markdown fences in relabeled outputs** (when `output_is_json: true`) → committee models sometimes wrap JSON in ```` ```json ... ``` ````. Teacher eval will then fail. Fix by downloading the train/test, stripping fences from `answer` fields, and re-uploading via `distil model upload-data`. See gotcha #3 in `references/tasks/upload-and-process-traces.md`.

---

## Step 5: Run Teacher Evaluation

Deterministic. Run directly.

**Before running, set expectations.** Tell the user the thresholds the report in Step 6 will apply, so they can override if their use case has different requirements:

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

This takes a few minutes. Poll until complete using the canonical pattern from `references/tasks/polling-jobs.md` (substitute `teacher-evaluation` as the status command, `sleep 60`). Do not proceed until status is `JOB_SUCCESS`.

---

## Step 6: Analyze Teacher Evaluation

Same analysis pattern as the dataset workflow's Step 4. The goal is a structured analysis that makes iteration decisions obvious.

### 6a. Gather data

1. Get aggregate metrics — **always use `--output json`**, the default text output omits LLM-as-a-Judge and other metrics:
```bash
distil model teacher-evaluation <model-id> --output json | jq '.aggregateMetrics'
```

2. Download per-example teacher predictions (see `references/tasks/retrieve-predictions.md` for full options):
```bash
distil model download-teacher-evaluation-predictions <model-id> --file-name teacher-predictions-iter-1.jsonl
```

Then load into a dataframe.

3. Read the uploaded data files (job_description.json, config.yaml) for context.

### 6b. Produce the Teacher Evaluation Analysis Report

Use the **Teacher Evaluation Analysis Report** template from `references/tasks/analyze-predictions.md`. The report produces a verdict (PROCEED / ITERATE / RETHINK).

Save the report with an iteration suffix (e.g., `teacher-eval-analysis-iter-1.md`) so subsequent iterations don't overwrite it. **After writing the report, tell the user**: the file path, the headline metric, and the verdict. Don't bury the report — it's the main output of this step and the user needs to see it to decide on next steps.

### 6c. Decision point

- **PROCEED** → Go to Step 8 (Confirm Before Training).
- **ITERATE** → Switch to `workflows/improving-a-model.md` → **Entry Point A** (ITERATE path — work through job description, data, synthgen/mutations, teacher model in that order). The report's "Recommended actions" section tells you which lever to start with. Trace-specific re-upload paths (`upload-traces`, `reprocess-traces`, `upload-data`) are covered there.
- **RETHINK** → Switch to `workflows/improving-a-model.md` → **Entry Point A** (RETHINK path — step back and question task type, task definition, judge instructions before touching individual levers).

**Also switch to Entry Point A if the teacher eval scores look questionable for any reason** — e.g. the judge flagged outputs as "bad" that look correct on inspection, or the metrics look inconsistent. The RETHINK path is the right place to confirm the judge is telling the truth before iterating on the model.

---

## Step 7: Iterate (return point)

This is the return point from `workflows/improving-a-model.md`. After each iteration loop completes there, come back to Step 6 to analyze the new teacher evaluation results. Repeat until the verdict is `PROCEED`.

---

## Step 8: Confirm Before Training

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

## Step 9: Train

Deterministic. Run only after the user confirms in Step 8.

```bash
distil model run-training <model-id>
```

Training takes several hours (typically 6+). The pipeline has three stages:
1. **Evaluate Teacher** — confirms teacher performance on the test set
2. **Generate Synthetic Data** — teacher generates training data from your seed examples + unstructured data
3. **Finetune Student** — student model is trained on the synthetic data, then evaluated against the test set

Poll until complete using the canonical pattern from `references/tasks/polling-jobs.md`. Swap `sleep 60` for `sleep 600` — training is multi-hour, so minute-scale polling is wasteful.

Suggest the user check back periodically. They can close the session and come back — `distil model training <model-id>` always shows current state.

Do not proceed to Step 10 until status is `JOB_SUCCESS`.

---

## Step 10: Analyze Training Results

Same analysis pattern as Step 6, but now comparing four models: the original production model, the base (untuned) student, the teacher, and the tuned student.

### 10a. Gather data

1. Get aggregate metrics — **always use `--output json`**, the default text output omits LLM-as-a-Judge and other metrics:
```bash
distil model training <model-id> --output json | jq '.aggregateMetrics'
```

2. Download tuned student predictions via the CLI:
```bash
distil model download-training-predictions <model-id> --file-name student-predictions-iter-1.jsonl
```

3. Download base student predictions (only available via API — see `references/tasks/retrieve-predictions.md`):
```python
response = requests.get(
    f"https://api.distillabs.ai/trainings/{training_job_id}/evaluation-results",
    headers=auth_header,
)
base_predictions_url = response.json()["base_student_evaluation_predictions_download_url"]
```

4. Also load:
   - **Teacher predictions** saved in Step 6
   - **Original model predictions** saved in Step 4

You now have four sets of predictions: **original production model**, **base student** (untuned baseline), **teacher** (upper bound), and **tuned student** (the trained model).

### 10b. Produce the Training Analysis Report

Use the **Training Analysis Report** template from `references/tasks/analyze-predictions.md`. Follow the **"Training from traces?"** note in Section 4 of the template — add an Original Model column to the metrics table. The most important comparison becomes Tuned vs. Original, since that's whether the trained student beats the production model it's meant to replace.

Save with an iteration suffix (e.g., `training-analysis-iter-1.md`). **After writing the report, tell the user**: the file path, the headline metric (tuned student primary score), the deltas vs. teacher and original model, and the verdict. The user needs this to decide whether to deploy or retune.

### 10c. Decision point

- **DEPLOY** → Go to Step 11. Tuned student is within ~0.05 of teacher AND beats the original model on the primary metric.
- **RETUNE** → Switch to `workflows/improving-a-model.md` → **Entry Point B** (retune levers). After retuning completes, return here and re-run Step 10 analysis.
- **ESCALATE** → Switch to `workflows/improving-a-model.md` → **Entry Point B** (ESCALATE section). Likely needs to go back to Entry Point A — the data, task definition, or distillation has a fundamental issue.

---

## Step 11: Deploy

Once satisfied with training results, deploy the model.

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
| Configuration | `references/configuration.md` |
| Upload + trace processing | `references/tasks/upload-and-process-traces.md` |
| Teacher evaluation | `references/tasks/teacher-evaluation.md` |
| Metrics interpretation | `references/evaluation-metrics.md` |
| Predictions download | `references/tasks/retrieve-predictions.md` |
| Analysis report templates | `references/tasks/analyze-predictions.md` |
| Polling long-running jobs | `references/tasks/polling-jobs.md` |
| Iteration (teacher eval or training) | `workflows/improving-a-model.md` |
| API authentication | `references/api-reference.md` |
| Training | `references/tasks/training.md` |
| Deployment | `references/tasks/deployment-integration.md` |
| Mutations (iteration) | `references/mutations-guide.md` |
