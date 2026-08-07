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
  train.jsonl / test.jsonl / traces.jsonl
  teacher-eval-analysis.md     # analyze-predictions writes here
  teacher-predictions.jsonl
  training-analysis.md         # after re-training, if applicable
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
- **Synthgen produces poor inputs even though teacher eval looks fine** (`question-answering` only) → `input_description` is incomplete. Move input structure out of `task_description` into `input_description`. For QA, synthgen reads `input_description` not `task_description` when generating inputs. Does NOT apply to other task types — for them, synthgen ignores `input_description` entirely; look at unstructured data, train examples, or the task-specific schema (`classes_description` / `tools`) instead.
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
- **Generation is slow** → raise `base.llm_num_parallel_requests` above the default of 4 (e.g. 8).

#### Lever 4 — Teacher model

Try this when the teacher is consistently wrong on complex reasoning, not just finicky on formatting.

See `references/model-catalog.md` for the teacher shortlist and constraints. Common moves:
- General task → try `zai.glm-5`.
- Coding-heavy → try `Qwen3-480B-A35B-Coder`.
- Multi-turn tool calling → most teachers support tool calling; a few are excluded (see `references/model-catalog.md`). Strong picks for quality: `zai.glm-5`, `Qwen3-235B-A22B-Instruct-2507`.

Do not flip teachers as the first move. A stronger teacher won't fix a vague job description.

### Re-upload loop

After any change:

Nothing is mutated in place. Every change means creating a **new** upload and a **new** teacher evaluation, which is what keeps earlier attempts available for comparison.

```bash
# 1. Create the next iteration dir and drop in a README describing what's being tested.
mkdir -p iteration-<N>
# Write iteration-<N>/README.md with 2-4 sentences on the lever being pulled.

# 2. Create a new upload from the revised data
distil upload create --data ./data-dir
# Output: Upload successful. Upload ID: <new-upload-id>
# (On the traces path, re-process instead — see Trace-specific paths below.)

# 3. Wait for it, then evaluate the teacher against the NEW upload id
distil upload status <new-upload-id> --output json | jq -r '.status'
distil teacher-evaluation create-from-upload <new-upload-id>
# Output: Teacher evaluation started. Teacher Evaluation ID: <new-teacher-evaluation-id>
```

**The failure mode to guard against is using the wrong ID.** Because nothing is overwritten, passing last iteration's `<upload-id>` succeeds — it just re-evaluates the old data and returns the old scores, and you waste time wondering why your changes had no effect. Read the new ID out of the command output; do not reuse a shell variable from the previous iteration. If you are unsure which upload an evaluation actually ran against, `distil teacher-evaluation show <teacher-evaluation-id>` reports its `upload_id` — that is the authoritative check, not your notes.

Append a log entry (`references/tasks/maintain-run-log.md`) after the upload succeeds, recording which lever was pulled and why.

For the polling loop, see `references/tasks/polling-jobs.md`. Then go back to the workflow you came from (Step 4 in dataset-to-model, Step 6 in traces-to-model) and analyze the new results — pass the current `iteration-<N>/` as the working directory to `references/tasks/analyze-predictions.md`.

### Trace-specific paths

If you're in the traces workflow, you have three options, cheapest last:

- **`distil traces upload --data <dir>`** then `distil upload create-from-traces <new-traces-id>` — needed only when the trace files themselves changed (new traces, or a curated `test.jsonl` added). Slowest.
- **`distil upload create-from-traces <traces-id> --config <file>`** (or `--job-description <file>`) — reuses the trace files already on the platform and re-runs processing with new parameters. Use this for processing-only changes; `<traces-id>` is in the run log from Step 3.
- **`distil upload create --data <dir>`** — upload a manually fixed dataset, e.g. after `distil upload download` and stripping markdown fences from relabeled answers. No reprocessing at all.

Each `create-from-traces` run produces a new `<upload-id>` and it is immediately usable — teacher evaluation reads an upload directly, so there is no download-and-re-upload hop. Just wait for processing to finish:

```bash
distil upload status <new-upload-id> --output json | jq -r '.status'   # wait for JOB_SUCCESS
distil teacher-evaluation create-from-upload <new-upload-id>
```

After either of the first two options, re-run the test-set approval gate (`references/tasks/test-set-approval.md`) and re-offer the uploads deep dive (`references/tasks/analyze-uploads.md`) before returning to teacher evaluation.

---

## Entry Point B: Training Wasn't Good Enough

The training analysis verdict was `RETUNE` or `ESCALATE`. The tuned student is below target — significantly below teacher, or doesn't beat the original production model (traces workflow).

The three levers from Entry Point A still apply (they require regenerating the training dataset). Two additional levers become available because the training dataset already exists and can be reused.

**There is no retune command.** Reusing generated data means round-tripping the training dataset through your machine with an edited config:

```bash
# 1. Download the dataset you already paid to generate
distil training-dataset download <training-dataset-id> --destination ./dataset

# 2. Edit ./dataset/config.yaml -- base.student_model_name and/or the tuning section

# 3. Create a new dataset from the edited files (no generation job, no generation cost)
distil training-dataset create --data ./dataset
# Output: Upload successful. Training Dataset ID: <new-training-dataset-id>

# 4. Train on it
distil slm create-from-training-dataset <new-training-dataset-id>
```

`download` writes the filenames `create` reads, so nothing needs renaming. The new dataset's `source` is `direct_upload` and its status is `JOB_SUCCESS` immediately, since no job runs.

**Caveat:** `distil training-dataset download` is credit-gated with 0 credits granted by default. If it returns a 402, this shortcut is unavailable and the fallback is a fresh `distil training-dataset create-from-upload <upload-id>` with an updated config on the upload — which pays the 2-credit generation cost again.

Step 4 starts a new credit-consuming training run. Never run it on your own initiative: confirm the chosen student and parameters with the user first, exactly as the "Confirm Before Training" gate requires.

#### Lever 5 — A different student model

The most effective lever when the student is close to but below the teacher. Change `base.student_model_name` in step 2 above.

See `references/model-catalog.md` for sizing and compatibility. Common escalations:
- Student was 1B and task needs more capacity → try 3B or 4B.
- Need tool calling → student family must be Qwen3 or Llama 3.
- Edge deployment constraint → go smaller (135M–350M) but expect a quality drop.

#### Lever 6 — Different tuning parameters

Useful when the student is right-sized but underfit or overfit. See `references/configuration.md` §2 for the full `tuning` parameter set.

- **Underfit** (tuned student barely beats base student) → increase `num_train_epochs`, consider enabling RLVR via `rlvr_dataset_size: 0.3`.
- **Overfit** (tuned student regressed on examples base got right) → lower `num_train_epochs`, reduce `generation_target` so synthgen doesn't drown the real examples, or add diversity via mutations.

**Tuning diagnostics — read the training logs.** `distil slm logs <slm-id>` returns the logs of the job that produced the SLM. If they do not include the detail you need (loss curves, per-epoch metrics), ask the user to paste what they can see; those numbers usually reveal the cause of a bad run. Check them before picking a tuning-parameter change:

| Symptom in logs | Likely cause | Lever |
|-----------------|--------------|-------|
| Loss drops to ~0 very quickly | Label leakage or near-duplicates in train | Fix upstream in data, not tuning — go back to Lever 2 |
| Loss plateau, never drops | Learning rate too low, or not enough epochs | Raise `num_train_epochs`, try a different `learning_rate_scheduler` |
| NaN loss mid-training | Numerical instability (often bf16 edge cases or bad examples) | Disable `bf16`, or re-check data for pathological inputs |
| Val-loss rising while train-loss keeps falling | Overfitting | Lower `num_train_epochs`, add diversity via mutations |

### When to ESCALATE instead of re-training

If the tuned student barely beats the base student despite good teacher scores, changing the student or the tuning parameters won't help — the distillation isn't transferring knowledge. Go back to Entry Point A and revisit task definition, data quality, and whether the task type itself is right.

`distil slm metrics <slm-id> --output json` is what tells you this: compare `.tuned_model_performance` against `.base_model_performance`. A small gap there, with a strong `.teacher_performance`, is the ESCALATE signal.

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
| Training dataset round-trip (`download` / `create`) | `references/cli-reference.md` (`## Training Datasets`) |
| Training and re-training | `references/tasks/training.md` |
| Trace-specific gotchas | `references/tasks/upload-and-process-traces.md` |
| Run log format and triggers | `references/tasks/maintain-run-log.md` |
| Test-set approval (traces) | `references/tasks/test-set-approval.md` |
| Uploads / data consistency analysis | `references/tasks/analyze-uploads.md` |
