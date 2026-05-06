# Job Description Guide

The `job_description.json` file is the most important input to the platform. It drives teacher evaluation, synthetic data generation, and the LLM-as-a-Judge scoring. Treat it with the same care as a production system prompt.

For the per-task-type structure (which fields are required for which task), see the relevant file under `references/tasks/prepare-data/`. This file covers what makes each field *good* regardless of task.

---

## The Fields

### `task_description` (required for all tasks)

What the model should do. This is read by:
- The **teacher** during evaluation (all task types).
- The **LLM-as-a-Judge** during scoring (all task types).
- **Synthgen** when generating training data — though its role differs by task:
  - For `question-answering`: synthgen uses `input_description` to generate inputs, and `task_description` to generate the corresponding outputs (answers).
  - For all other task types: `task_description` drives generation of full input/output pairs (combined with the task-specific schema and unstructured data — see the `input_description` section below for the per-task breakdown).

A good `task_description`:
- Describes the output format precisely, with an inline example if the output is structured (JSON, labels, tool calls).
- Lists what to include AND what to exclude.
- Covers edge cases the model will see in production (missing fields, ambiguous inputs, multi-language, etc.).
- Reads like a specification, not vague guidance.

A bad `task_description`:
- "Extract useful information." — no format, no criteria.
- "Be concise." — undefined.
- Only covers the happy path, leaving the teacher to guess on edge cases.

### `input_description` (used only by `question-answering`)

What the input looks like. **Only the `question-answering` task type uses this field** — synthgen reads it to generate new question/answer pairs, where `input_description` drives the inputs and `task_description` drives the outputs. The platform validates its presence for `question-answering` and will reject uploads without it.

For all other task types, synthgen does not read `input_description`. Inputs are generated from a different combination of fields per task:

| Task | What synthgen uses to generate inputs |
|------|---------------------------------------|
| `classification` | `task_description` + `classes_description` + unstructured data + train examples |
| `tool-calling-closed-book` | `task_description` + `tools` schema + unstructured data + train examples |
| `multi-turn-tool-calling-closed-book` | `task_description` + `tools` schema + unstructured data + train examples |
| `question-answering-closed-book` | `task_description` + unstructured `context` passages (the knowledge source) |
| `question-answering-open-book` | `task_description` + unstructured `context` passages |

You can omit `input_description` for these tasks, or include it for documentation — it will not change synthgen behavior.

For `question-answering` specifically, `input_description` is **self-contained**: synthgen sees it without `task_description`. If the input structure is only described in `task_description`, synthgen will produce poor inputs even if teacher evaluation (which uses `task_description`) looks fine.

A good `input_description` (for `question-answering`):
- Describes markers and sections in the input ("Invoice text starting with 'Invoice #', followed by line items, then totals").
- Lists format variations and noise patterns.
- Gives at least one concrete example.

### `llm_as_a_judge_instructions` (optional but high-leverage)

Binary pass/fail criteria that the judge LLM applies. A vague judge prompt makes teacher evaluation noisy — you'll chase scores that reflect judge error, not model error.

A good `llm_as_a_judge_instructions`:
- Defines what checks must pass for the output to be "good".
- Names specific fields or values that must match.
- Explicitly says what can be ignored (whitespace, casing, ordering, paraphrasing).

If teacher scores look low but spot-checks show the answers are correct, fix the judge prompt before blaming the model.

### Task-specific fields

| Task | Additional fields |
|------|-------------------|
| `classification` | `classes_description` — map of class name → description |
| `tool-calling-closed-book` | `tools` — array of JSON Schemas in OpenAI function-calling format |
| `multi-turn-tool-calling-closed-book` | `tools` — same format as single-turn |
| `question-answering-open-book` | None beyond the common ones |
| `question-answering-closed-book` | None beyond the common ones |

See `references/tasks/prepare-data/<task-type>.md` for worked examples of each.

---

## Iteration

You will almost always revise the job description at least once after teacher evaluation. Common revisions:

- **Teacher consistently misses entity type X** → add explicit handling of X to `task_description`.
- **Synthgen produces poor inputs but teacher eval is fine** (`question-answering` only) → `input_description` is incomplete; move input structure details out of `task_description` into `input_description`. For other task types, this symptom usually points at unstructured data or train examples, not the job description.
- **Judge scores look too harsh** → add ignore rules to `llm_as_a_judge_instructions` (whitespace, casing, paraphrasing).
- **Judge scores look too lenient** → tighten the judge criteria.

After any revision, you must re-upload the data — `job_description.json` cannot be patched in place. See the re-upload verification in `workflows/improving-a-model.md`.

---

## Common Pitfalls

1. **(`question-answering` only) Putting input details only in `task_description`.** For QA, synthgen reads `input_description` not `task_description` when generating inputs, so the training inputs lose structure. Move input structure into `input_description`. Does NOT apply to other task types — for them, `input_description` is unused.
2. **Vague judge instructions.** `"Output 'good' if correct, else 'bad'"` leaves the judge to define "correct". Spell it out.
3. **Missing edge cases.** The teacher generalizes from what you describe. If your prod traffic has missing fields, describe what to do with them.
4. **Inconsistency with training data.** If `task_description` says "return only the value, no explanation" but half your training answers have explanations, the teacher learns the wrong thing. Lint your train data against your description.
