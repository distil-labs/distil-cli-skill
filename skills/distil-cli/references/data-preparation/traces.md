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

## From an inference endpoint

A user with no trace file can have the platform collect one. A distil labs inference endpoint is
an OpenAI-compatible gateway that fronts the model they already run in production and keeps a
copy of every call it serves. Setting one up and downloading from it is an execution-backend
operation: `../execution/cli.md` § Inference endpoints (collecting traces), or the section of the
same name in `../execution/backend-api.md`.

What the download writes is the platform's record of each call: identifiers, timings and
metadata around the request and the response. It is not the observation format above, so convert
it before running trace processing on it.

Never assume the field names. Look at one record first:

```bash
head -1 <endpoint-name>-traces.jsonl | jq 'keys'
head -1 <endpoint-name>-traces.jsonl | jq '{input, output}'   # adjust to the keys you saw
```

Then write a conversion that, per line, takes the request's messages and the assistant reply the
endpoint returned and emits one `{"messages": [...]}` object in the chosen
`observation_format`. Check the first converted line by eye and the line count against the
source before uploading anything, and apply the conversion guidance below as for any other
source.

Two things to expect in endpoint records that a hand-written trace file does not have:

- **The system prompt is in every record**, because the endpoint saw the real request.
  `remove_system_prompt_from_traces` (default true) strips it, and its content belongs in the job
  description instead.
- **Failed and empty calls are recorded too.** Drop records with no assistant reply rather than
  emitting a conversation that ends on the user's turn.

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
