# Job Description Guide

The `job_description.json` file is the most important input to the platform. It drives teacher evaluation, synthetic data generation, and the LLM-as-a-Judge scoring. Treat it with the same care as a production system prompt.

For the per-task-type structure (which fields are required for which task), see the relevant file under `references/tasks/prepare-data/`. This file covers what makes each field *good* regardless of task.

---

## The Fields

### `task_description` (required for all tasks)

What the model should do. This is read by the teacher during evaluation, and by the LLM-as-a-Judge during scoring.

A good `task_description`:
- Describes the output format precisely, with an inline example if the output is structured (JSON, labels, tool calls).
- Lists what to include AND what to exclude.
- Covers edge cases the model will see in production (missing fields, ambiguous inputs, multi-language, etc.).
- Reads like a specification, not vague guidance.

A bad `task_description`:
- "Extract useful information." — no format, no criteria.
- "Be concise." — undefined.
- Only covers the happy path, leaving the teacher to guess on edge cases.

### `input_description` (required for `question-answering`, recommended elsewhere)

What the input looks like. This field is **self-contained**: the synthgen model that generates training data sees only `input_description`, not `task_description`. If the input structure is only described in `task_description`, synthgen will produce poor data even if teacher evaluation looks fine.

A good `input_description`:
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
- **Synthgen produces poor data but teacher eval is fine** → `input_description` is incomplete; move input structure details out of `task_description` into `input_description`.
- **Judge scores look too harsh** → add ignore rules to `llm_as_a_judge_instructions` (whitespace, casing, paraphrasing).
- **Judge scores look too lenient** → tighten the judge criteria.

After any revision, you must re-upload the data — `job_description.json` cannot be patched in place. See the re-upload verification in `workflows/improving-a-model.md`.

---

## Common Pitfalls

1. **Putting input details only in `task_description`.** Synthgen can't see `task_description`, so the training data loses structure. Move input structure into `input_description`.
2. **Vague judge instructions.** `"Output 'good' if correct, else 'bad'"` leaves the judge to define "correct". Spell it out.
3. **Missing edge cases.** The teacher generalizes from what you describe. If your prod traffic has missing fields, describe what to do with them.
4. **Inconsistency with training data.** If `task_description` says "return only the value, no explanation" but half your training answers have explanations, the teacher learns the wrong thing. Lint your train data against your description.
