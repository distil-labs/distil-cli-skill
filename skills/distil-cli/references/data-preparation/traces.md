# Data Preparation: Traces

Input for the trace processing stage.

## Trace directory

```
traces-input/
├── config.yaml           # include a trace_processing section (../configuration.md)
├── job_description.json  # task definition (../job-description.md)
├── traces.jsonl
└── test.jsonl            # optional curated test set; skips the generated test split
```

## Observation formats (`trace_processing.observation_format`)

**`openai_messages`** (default): each line has a `messages` array, with roles `system`,
`user`, `assistant` and `tool`. Assistant turns can carry `tool_calls`.

```jsonl
{"messages": [{"role": "system", "content": "..."}, {"role": "user", "content": "..."}, {"role": "assistant", "content": "..."}]}
```

**`openai_messages_with_images`**: the same, but user content can be a list of
`{"type": "text", ...}` / `{"type": "image_url", "image_url": {"url": ...}}` parts.

**`unstructured_with_openai_messages`**: each line has a `context` field whose string wraps
a messages array. Use it for traces doubling as unstructured context.

## Conversion guidance

- Constant prompt parts belong in the job description. `remove_system_prompt_from_traces`
  (default true) strips leading system messages anyway. Variable input → user message, model
  output → assistant message.
- Every trace is a multi-turn conversation rewritten whole. A simple exchange is a two-turn
  conversation.
- `synthgen.validation_max_total_length` (default 30,000 chars) applies to processed
  examples. If inputs embed documents or schemas, raise it.
- Emit clean JSONL: strip control characters and unicode line separators (U+2028/U+2029).

Any active task type can train from traces. For RAG-style traces with retrieved context
embedded in the user message, use `question-answering` (context stays inside `question`).
