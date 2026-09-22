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
same name in `../execution/backend-api.md`. `../../workflows/endpoint-to-model.md` sequences it.

The download is one record per call, and a record is not the observation format above. The
fields that matter:

| Field | What it holds |
|---|---|
| `input` | The request body as a JSON string: `model`, `messages`, and `tools` when the caller sent any |
| `output` | The chat completions response as a JSON string; the reply is `choices[0].message` |
| `metadata.status` | The HTTP status the caller received |
| `metadata.source` | Which model answered: `fallback`, or `primary` when a trained model is fronting the endpoint |
| `start_time`, `latency` | When the call started and how long it took, in seconds |

The rest is identifiers and platform bookkeeping. Inspect one record before converting:

```bash
head -1 <endpoint-name>-traces.jsonl | jq '{status: .metadata.status, source: .metadata.source, input: (.input | fromjson | keys), output: (.output | fromjson | .choices[0].message | keys)}'
```

Conversion, per line: parse `input` and `output`, append `choices[0].message` as the assistant
turn to the request's `messages`, carry `tools` across when present, and write one
`{"messages": [...]}` object. Skip the records that should not become training data:

- **Failed calls.** Keep `metadata.status == 200` only.
- **Empty replies.** Skip a response with neither `content` nor `tool_calls`, rather than
  emitting a conversation that ends on the user's turn.
- **The wrong model's answers**, once a student fronts the endpoint. Filter on
  `metadata.source` to keep the fallback's answers, the student's, or both, depending on what
  the iteration is meant to learn from.

```python
import json

def convert(record):
    if record["metadata"].get("status") != 200:
        return None
    request = json.loads(record["input"])
    reply = json.loads(record["output"])["choices"][0]["message"]
    if not reply.get("content") and not reply.get("tool_calls"):
        return None
    assistant = {"role": "assistant", "content": reply.get("content") or ""}
    if reply.get("tool_calls"):
        assistant["tool_calls"] = reply["tool_calls"]
    converted = {"messages": [*request["messages"], assistant]}
    if request.get("tools"):
        converted["tools"] = request["tools"]
    return converted

with open("<endpoint-name>-traces.jsonl") as src, open("traces.jsonl", "w") as dst:
    for line in src:
        converted = convert(json.loads(line))
        if converted is not None:
            dst.write(json.dumps(converted, ensure_ascii=False) + "\n")
```

Check the first converted line by eye and the kept count against the source before uploading
anything, and apply the conversion guidance below as for any other source. Two things endpoint
records have that a hand-written trace file does not:

- **The system prompt is in every record**, because the endpoint saw the real request.
  `remove_system_prompt_from_traces` (default true) strips it, so its content belongs in the job
  description's `task_description` instead, and the two must say the same thing.
- **The reply can carry `reasoning`** next to `content` when the fallback is a reasoning model.
  The conversion above keeps `content` only, which is what the caller's application used.

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
