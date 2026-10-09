# Job Description

`job_description.json` defines the task for the teacher, synthgen, and the judge. Unknown
fields are rejected at validation.

## Fields by task type

**Classification**:

```json
{
  "task_description": "...",
  "classes_description": {"class_name": "when this class applies", "...": "..."},
  "synthetic_data_generation_instructions": "...",
  "trace_processing_instructions": "..."
}
```

**Question answering** (`question-answering`):

```json
{
  "task_description": "...",
  "llm_as_a_judge_instructions": "...",
  "synthetic_data_generation_instructions": "...",
  "trace_processing_instructions": "..."
}
```

**Conversation** (`chat-completion`, `chat-completion-agentic`):

```json
{
  "task_description": "...",
  "tools": [{"type": "function", "function": {"name": "...", "description": "...",
             "parameters": {"type": "object", "properties": {...}, "required": [...]}}}],
  "llm_as_a_judge_instructions": "...",
  "synthetic_data_generation_instructions": "...",
  "trace_processing_instructions": "..."
}
```

`tools` is the OpenAI function-calling spec, with unique names. It is REQUIRED for
`chat-completion-agentic`. `chat-completion` may omit the field (or pass null or `[]`) for a
tool-free conversational model. Parameter schemas can declare `default` values.
`tool_call_equivalence` treats an argument at its default as equal to omitting it. Data rules
for tools: `data-preparation/chat-completion.md`.

Every field except `task_description` (and `classes_description` / `tools` where shown) is
optional. There is no system-prompt field: `task_description` is rendered into the system prompt
the model trains and serves with, so write it as one.

## What each field feeds

- `task_description`: in every teacher prompt (eval, synthgen, judge). Derive it from the
  system prompt the user's production system runs, with the same care: output format with an
  example, include/exclude rules, edge cases. It must stay matched to that production
  prompt and unchanged across iterations: it defines what a correct answer is for evaluation
  and the judge too. To improve results, change the teacher or the data, not this field.
- `classes_description` and `tools`: define the label and call space. Their descriptions
  directly shape generated data.
- `synthetic_data_generation_instructions` (all task types): extra guidance injected into every
  synthetic-data-generation prompt. Use it to describe the generated inputs: formats, domains,
  variation, noise. For a reasoning student, also `reasoning-models.md`.
- `llm_as_a_judge_instructions` (every task type except classification, where the create fails
  if it is sent; classification is scored by label accuracy): the instructions the judge model
  is given when it scores a prediction, for both judge metrics (`evaluation-metrics.md` § Metrics
  by task). State pass/fail criteria: what a correct answer must contain, what to ignore (order,
  whitespace, paraphrasing). A workable shape is "Output 'good' if the prediction correctly
  completes the task, otherwise output 'bad'", then the criteria that decide it, including format
  rules (for example, no code fences). Vague criteria make every downstream verdict noisy. When
  the judge mismeasures, update these instructions or use a stronger judge model; either is a
  measurement change, so earlier scores are not comparable.
- `trace_processing_instructions` (optional, all task types): task-specific guidance appended
  to the relabelling rewrite and fix instructions only. It is unused outside relabelling
  (`../stages/relabel-traces.md`). Use it when the edits must respect something unusual about the traces, for
  example "this is a live phone call; preserve the caller's interruptions and any cut-off
  utterances verbatim". Leave it out when no special handling is needed.
