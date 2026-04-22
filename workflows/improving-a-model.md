# Workflow: Improving a Model

Use this workflow whenever results aren't good enough — either teacher evaluation scores below the thresholds, or a completed training that doesn't deploy. Both the `dataset-to-model` and `traces-to-model` workflows point here for iteration.

This is a **single workflow** with two distinct entry points because the levers are largely the same. Read the matching section for your situation, but skim both — the levers overlap.

---

## Iteration Discipline

An **iteration** is one full cycle: prep data → upload → run {trace processing, teacher evaluation, training} → analyze. This file owns the `iteration-N/` convention; reference files (including `references/tasks/analyze-predictions.md`) just accept a working directory.

### One `iteration-N/` directory per attempt

Create a new `iteration-N/` directory at the project root before making any change. The directory scopes everything for that attempt, so filenames inside don't need iteration suffixes.

```
iteration-2/
  README.md                    # "what's being tested" in 2-4 sentences
  job_description.json         # only files that changed from iter-1
  config.yaml
  train.csv / test.csv / traces.jsonl
  teacher-eval-analysis.md     # analyze-predictions writes here
  teacher-predictions.jsonl
  training-analysis.md         # after retune, if applicable
  student-predictions.jsonl
  upload-consistency.md        # if Analyze Uploads was run
  original-model-analysis.md   # traces workflow, via test-set-approval
```

Any task that produces a report is **given** the iteration directory as its working-dir parameter — the task file does not decide where the report lives.

### Change one lever at a time

If you change the job description AND the data AND the teacher model simultaneously, you won't know what helped. Pull one lever per iteration. The `iteration-N/` directory only needs to contain the files that changed.

### Append to the run log

At the start of each iteration, append a "start" entry to `model-building-log-<name>.md` naming the lever being tested. At the end of the iteration (after the analysis report is written), append an "end" entry with the verdict delta from the prior iteration. See `references/tasks/maintain-run-log.md`.

### Token-burn awareness

At iteration #3 or later, remind the user that each iteration costs: re-upload credits + teacher-evaluation credits + Claude-side analysis tokens. Before starting iteration #3+, confirm the lever plan with the user rather than racing into another attempt.

---

## Entry Point A: Teacher Evaluation Wasn't Good Enough

The analysis report verdict was `ITERATE` or `RETHINK`. The teacher failed to solve the task on your test set — so the student won't either. You must fix the inputs before training.

### If the verdict is RETHINK (primary metric < 0.60)

Before touching any lever, step back and confirm with the user:

- **Is the task type right?** e.g. a "tool calling" task with free-text tool arguments should probably be `question-answering` with a JSON output. A "classification" task with overlapping classes should probably be reduced to a cleaner taxonomy first. See `references/task-selection-guide.md`.
- **Is the task well-defined?** Can the user articulate, in one sentence, what distinguishes a "good" output from a "bad" one? If not, the `llm_as_a_judge_instructions` will be weak and teacher eval scores will be unreliable — the score reflects judge noise, not model skill.
- **Does the job description capture what you actually want?** Read `task_description` aloud. Would a human contractor with no context produce outputs matching your expectations? If no, the teacher can't either.
- **Is the judge the problem?** Pull a handful of incorrect predictions from the analysis report. If they look correct to you but the judge scored them bad, fix `llm_as_a_judge_instructions` before anything else — the teacher eval number is lying.

Only once those fundamentals are settled, work through the levers below.

### If the verdict is ITERATE (primary metric 0.60–0.80 text, < 0.70 tool calling)

Work through the levers top-down. Most iterations end at Lever 1 or 2.

#### Lever 1 — Job description

Most common fix. Read `references/job-description-guide.md` for what makes each field good.

- **Teacher misses entity type X** → add explicit handling to `task_description`.
- **Synthgen produces poor data even though teacher eval looks fine** → `input_description` is incomplete. Move input structure out of `task_description` into `input_description`. Synthgen does NOT see `task_description`, only `input_description`.
- **Judge seems too strict** → relax `llm_as_a_judge_instructions` (ignore whitespace, casing, paraphrasing).
- **Judge seems too lenient** → tighten judge criteria.

#### Lever 2 — Training/test data

- Add manually curated examples covering the failure patterns surfaced in the analysis report.
- Remove or fix examples with ambiguous or incorrect labels.
- For classification: ensure every class in `classes_description` appears in train data.
- For tool calling: verify every `answer` parses as valid JSON (markdown fences in relabeled traces are a common culprit — see `references/tasks/upload-and-process-traces.md` gotcha #3).

#### Lever 3 — Synthgen parameters and mutations

Only touch these after Levers 1 and 2 have plateaued. See `references/configuration.md` §4 for the full parameter set and `references/mutations-guide.md` for how mutations work.

- **Data lacks diversity on scenarios you care about** → add `synthgen.mutation_topics` targeting those scenarios. Use 1–2 topic lists of 3–10 topics each.
- **Synthgen produces too many near-duplicates of seed data** → lower `synthgen.validation_similarity_threshold` (default 0.95 — try 0.90).
- **Synthgen output length is wrong** → swap the built-in mutator in `basic_mutators_to_use` (default `["complexity"]` — try `["length"]` or `["specificity"]`). Use at most one built-in mutator at a time.
- **Generating too few/too many examples** → adjust `generation_target` (default 10,000).
- **Generation is slow** → set `parallel_llm_calls: true`.

#### Lever 4 — Teacher model

Try this when the teacher is consistently wrong on complex reasoning, not just finicky on formatting.

See `references/model-catalog.md` for the teacher shortlist and constraints. Common moves:
- General task → try `zai.glm-5` or `deepseek.v3.2`.
- Coding-heavy → try `Qwen3-480B-A35B-Coder`.
- Multi-turn tool calling → teacher must be one of the 10 tested models (see `references/model-catalog.md`). Strong picks for quality: `zai.glm-5`, `deepseek.v3.2`, `Qwen3-235B-A22B-Instruct-2507`.

Do not flip teachers as the first move. A stronger teacher won't fix a vague job description.

### Re-upload loop

After any change:

```bash
# 1. Create the next iteration dir and drop in a README describing what's being tested.
mkdir -p iteration-<N>
# Write iteration-<N>/README.md with 2-4 sentences on the lever being pulled.

# 2. Capture current upload ID so we can confirm a new one is created
old_upload_id=$(distil model show <model-id> --output json | jq -r '.upload_ids[0] // "none"')

# 3. Re-upload — creates a new upload, doesn't overwrite
distil model upload-data <model-id> --data ./data-dir
# (Or distil model upload-traces for the traces path.)

# 4. Confirm a NEW upload was created — if IDs match, re-upload didn't take
new_upload_id=$(distil model show <model-id> --output json | jq -r '.upload_ids[0]')
if [ "$new_upload_id" = "$old_upload_id" ]; then
    echo "ERROR: upload ID unchanged. Re-upload didn't take. Stop and investigate."
    exit 1
fi
echo "New upload: $new_upload_id (was $old_upload_id)"

# 5. Re-run teacher evaluation (uses the latest upload automatically)
distil model run-teacher-evaluation <model-id>
```

The upload ID check matters because silent failures here are the worst kind — you'll re-run teacher evaluation against the *old* job description and get the *old* results, then waste time wondering why your changes had no effect.

Append a log entry (`references/tasks/maintain-run-log.md`) after the re-upload succeeds, recording which lever was pulled.

For the polling loop, see `references/tasks/polling-jobs.md`. Then go back to the workflow you came from (Step 4 in dataset-to-model, Step 6 in traces-to-model) and analyze the new results — pass the current `iteration-<N>/` as the working directory to `references/tasks/analyze-predictions.md`.

### Trace-specific paths

If you're in the traces workflow, you have three re-upload options:

- **`distil model upload-traces`** — re-runs full trace processing including committee relabeling. Slowest.
- **`distil model reprocess-traces`** — skips re-uploading the trace files, just re-runs processing with new `trace_processing` params. Use this for processing-only changes.
- **`distil model upload-data`** — upload a manually fixed dataset, e.g. after stripping markdown fences from relabeled answers.

After any of the first two, re-run the test-set approval gate (`references/tasks/test-set-approval.md`) and re-offer the uploads deep dive (`references/tasks/analyze-uploads.md`) before returning to teacher evaluation.

---

## Entry Point B: Training Wasn't Good Enough

The training analysis verdict was `RETUNE` or `ESCALATE`. The tuned student is below target — significantly below teacher, or doesn't beat the original production model (traces workflow).

The three levers from Entry Point A still apply (they require re-training from scratch). Two additional levers become available because the synthetic dataset already exists.

#### Lever 5 — Retune with a different student

Retune creates a new model based on the synthetic data from a prior training run — no re-upload or re-generation needed. This is the most effective lever when the student is close to but below the teacher.

See `references/model-catalog.md` for sizing and compatibility. Common escalations:
- Student was 1B and task needs more capacity → try 3B or 4B.
- Need tool calling → student family must be Qwen3 or Llama 3.
- Edge deployment constraint → go smaller (135M–350M) but expect a quality drop.

```bash
distil model retune <model-id> \
  --name <new-name> \
  --student-model <new-student> \
  --tuning-parameters <file>
```

Or pass a full config with `--config` — only the `tuning` section is used. See `references/cli-reference.md` for flag details.

#### Lever 6 — Retune with different tuning parameters

Useful when the student is right-sized but underfit or overfit. See `references/configuration.md` §2 for the full `tuning` parameter set.

- **Underfit** (tuned student barely beats base student) → increase `num_train_epochs`, consider enabling RLVR via `rlvr_dataset_size: 0.3`.
- **Overfit** (tuned student regressed on examples base got right) → lower `num_train_epochs`, reduce `generation_target` so synthgen doesn't drown the real examples, or add diversity via mutations.

**Tuning diagnostics — ask for training-job logs.** The CLI doesn't expose training-job logs (loss curves, per-epoch metrics) directly, but if the user can paste them into the conversation they usually reveal the cause of a bad run. Ask for them before picking a tuning-parameter change:

| Symptom in logs | Likely cause | Lever |
|-----------------|--------------|-------|
| Loss drops to ~0 very quickly | Label leakage or near-duplicates in train | Fix upstream in data, not tuning — go back to Lever 2 |
| Loss plateau, never drops | Learning rate too low, or not enough epochs | Raise `num_train_epochs`, try a different `learning_rate_scheduler` |
| NaN loss mid-training | Numerical instability (often bf16 edge cases or bad examples) | Disable `bf16`, or re-check data for pathological inputs |
| Val-loss rising while train-loss keeps falling | Overfitting | Lower `num_train_epochs`, add diversity via mutations |

### When to ESCALATE instead of retune

If the tuned student barely beats the base student despite good teacher scores, retuning won't help — the distillation isn't transferring knowledge. Go back to Entry Point A and revisit task definition, data quality, and whether the task type itself is right.

---

## Reference Files

| Topic | Reference |
|-------|-----------|
| Job description authoring | `references/job-description-guide.md` |
| Model catalog + compatibility + recommendations | `references/model-catalog.md` |
| All config parameters | `references/configuration.md` |
| Mutation semantics and patterns | `references/mutations-guide.md` |
| Metric interpretation | `references/evaluation-metrics.md` |
| Analysis report templates | `references/tasks/analyze-predictions.md` |
| Polling loop | `references/tasks/polling-jobs.md` |
| Retune command | `references/cli-reference.md` (see `distil model retune`) |
| Trace-specific gotchas | `references/tasks/upload-and-process-traces.md` |
| Run log format and triggers | `references/tasks/maintain-run-log.md` |
| Test-set approval (traces) | `references/tasks/test-set-approval.md` |
| Uploads deep dive (optional) | `references/tasks/analyze-uploads.md` |
