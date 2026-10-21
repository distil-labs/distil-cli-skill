# Job Description

`job_description.json` defines the task for the teacher, synthgen, and the judge. Unknown
fields are rejected at validation.

## Fields by task type

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

Every field except `task_description` (and `tools` where shown) is optional. There is no
system-prompt field: `task_description` is rendered into the system prompt the model trains and
serves with, so write it as one. With `base.variable_system_prompt` on, each row carries its own
system prompt instead (`variable-system-prompts.md`).

## What each field feeds

- `task_description`: in every teacher prompt (eval, synthgen, judge). Derive it from the
  system prompt the user's production system runs, with the same care: output format with an
  example, include/exclude rules, edge cases. Examples in it come from the train split or from
  rows the user wrote for that purpose, never from `test.jsonl`. Keep them few and short: they
  become part of the system prompt the model trains and serves with.
  It defines what a correct answer is for evaluation and the judge too, so keep it fixed once
  the test set is locked and scored. Before that, and whenever the teacher misses conventions
  the user's labels follow (format, label names, length), write those conventions into it: it
  is often the biggest lever on teacher quality. A change after the first teacher evaluation
  needs the user's approval and a new teacher evaluation.
  With `base.variable_system_prompt` on, it states only what every application's system
  prompt has in common (`variable-system-prompts.md` § What `task_description` becomes).
- `tools`: defines the call space. Its descriptions directly shape generated data.
- `synthetic_data_generation_instructions` (all task types): extra guidance injected into every
  synthetic-data-generation prompt. Use it to describe the generated inputs: formats, domains,
  variation, noise. For a reasoning student, also `reasoning-models.md`.
- `llm_as_a_judge_instructions` (all task types): the instructions the judge model
  is given when it scores a prediction, for both judge metrics (`evaluation-metrics.md` § Metrics
  by task) and for the RLVR reward (`configuration.md`). State the criteria: what a correct answer
  must contain, what to ignore (order, whitespace, paraphrasing), including format rules (for
  example, no code fences). Vague criteria make every downstream verdict noisy. When the judge
  mismeasures, update these instructions or use a stronger judge model; either is a measurement
  change, so earlier scores are not comparable. The judge writes its score in a `<score>` tag, in
  one of two modes (§ Judge score modes).
- `trace_processing_instructions` (optional, all task types): task-specific guidance appended
  to the relabelling rewrite and fix instructions only. It is unused outside relabelling
  (`../stages/relabel-traces.md`). Use it when the edits must respect something unusual about the traces, for
  example "this is a live phone call; preserve the caller's interruptions and any cut-off
  utterances verbatim". Leave it out when no special handling is needed.

## Judge score modes

- **Pass/fail verdict (default).** Unless the instructions ask for something else, the judge
  answers `<score>good</score>` or `<score>bad</score>`, scored 1 or 0. A workable shape is
  "Output 'good' if the prediction correctly completes the task, otherwise output 'bad'", then
  the criteria that decide it. Use it when an answer is either correct or not. The aggregated
  metric is the fraction of predictions marked good.
- **Graded score.** When the instructions ask for a number from 0 to 1, the judge gives partial
  credit. Show the exact format, for example "Check the prediction against these four criteria:
  (1) ... (4) .... Output the fraction of criteria met as a number between 0 and 1 inside the
  score tag, for example <score>0.75</score>". Use it when an answer can be partly correct, for
  example against several independent criteria, and for RLVR when the student passes few
  rollouts: with pass/fail most rollout groups all score 0 and RLVR has nothing to rank inside
  the group. The aggregated metric is the mean score, not comparable with an earlier pass rate.

How the score is read:

- Exactly one `<score>` tag. A reply without a tag is read as a whole; several tags are
  unparseable.
- `good`, `yes`, `true`, `correct`, `pass`, `valid`, `1` read as 1; `bad`, `no`, `false`,
  `incorrect`, `wrong`, `fail`, `invalid`, `0` read as 0. Case-insensitive.
- Any other value is read as a number and clamped to 0-1: `8` from a 1-10 scale reads as 1, so
  always ask for 0-1. Percentages (`75%`) and fractions (`3/4`) are unparseable.
- Unparseable replies count as 0 in evaluation and are left out of the reward in RLVR.
