# Job Description

`job_description.json` defines the task for the teacher, synthgen, and the judge. Unknown
fields are rejected at validation.

## Fields by task type

**Classification**:

```json
{
  "task_description": "...",
  "classes_description": {"class_name": "when this class applies", "...": "..."},
  "trace_processing_instructions": "..." # Only relevant for trace processing
}
```

`synthetic_data_generation_instructions` is also accepted here and steers generation as it
does for the other task types. But `llm_as_a_judge_instructions` is NOT valid for
classification (the create fails); classification is judged by label accuracy anyway.

**QA family** (`question-answering`, `-open-book`, `-closed-book`):

```json
{
  "task_description": "...",
  "llm_as_a_judge_instructions": "...",
  "synthetic_data_generation_instructions": "...", # Optional, valid for every task type
  "trace_processing_instructions": "..." # Only relevant for trace processing
}
```

`llm_as_a_judge_instructions` is optional.

**Tool calling** (both tasks):

```json
{
  "task_description": "...",
  "tools": [{"type": "function", "function": {"name": "...", "description": "...",
             "parameters": {"type": "object", "properties": {...}, "required": [...]}}}],
  "llm_as_a_judge_instructions": "...",
  "trace_processing_instructions": "..." # Only relevant for trace processing
}
```

`tools` is OpenAI function-calling spec: at least one tool, unique names. Parameter schemas
may declare `default` values; `tool_call_equivalence` treats an argument at its default as
equal to omitting it.

## What each field feeds

- `task_description` — in every teacher prompt (eval, synthgen, judge). Derive it from the
  system prompt the user's production system runs, with the same care: output format with an
  example, include/exclude rules, edge cases. It must stay compliant with that production
  prompt and constant across iterations; it is not a tuning lever (pick a better teacher or
  fix data instead).
- `classes_description` / `tools` — define the label/call space; their descriptions directly
  shape generated data.
- `synthetic_data_generation_instructions` (optional, all task types) — extra guidance
  injected into every synthetic-data-generation prompt. Use it to describe what generated
  inputs should look like: formats, domains, variation, noise.
- `llm_as_a_judge_instructions` (optional, every task type except classification) — the
  instructions the judge model is given when it scores a prediction against the reference.
  State pass/fail criteria: what must match, what to ignore (order, whitespace,
  paraphrasing). A workable shape is "Output 'good' if the prediction matches the reference
  or is semantically equivalent, otherwise output 'bad'", then the criteria that decide it.
  Vague criteria make every downstream verdict noisy.
- `trace_processing_instructions` (optional, all task types) — task-specific guidance
  appended to the trace-processing rewrite and fix instructions only; unused outside trace
  processing. Use it when the edits must respect something unusual about the traces, e.g.
  "this is a live phone call; preserve the caller's interruptions and any cut-off utterances
  verbatim". Leave it out when no special handling is needed.
