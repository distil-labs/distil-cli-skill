# Data Preparation: Traces

Input for the test set from traces and trace processing stages.

## Trace directory

```
traces-input/
├── config.yaml           # trace_processing and traces_to_test_set sections (../configuration.md)
├── job_description.json  # task definition (../job-description.md)
├── traces.jsonl
└── test.jsonl            # optional curated test set; test set from traces adds to it
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

**`langfuse`**: the records of a distil labs inference endpoint, as a traces object created
directly from the endpoint stores them (`../execution/cli.md` § Traces from an endpoint). Never
written by hand.

## From an inference endpoint

An inference endpoint's records become a traces object either directly, with
`observation_format: langfuse` and no conversion, or by download, conversion and upload. When
to use which: `../inference-endpoints.md` § From records to a traces object. This section is the
conversion for the download route (`../execution/cli.md` § Download the traces, or
`../execution/backend-api.md` § Read the traces).

A downloaded record is one call, not an observation format. The fields that matter:

| Field | What it holds |
|---|---|
| `input` | The request body as a JSON string: `model`, `messages`, and `tools` when the caller sent any |
| `output` | The chat completions response as a JSON string; the reply is `choices[0].message` |
| `metadata.status` | The HTTP status the caller received |
| `metadata.source` | Which model answered: `fallback`, or `primary` when a trained model is fronting the endpoint |
| `start_time`, `latency` | When the call started and how long it took, in seconds |

The rest is identifiers. Read one record before converting. Conversion, per line: parse `input` and `output`, append `choices[0].message` as the assistant
turn to the request's `messages`, carry `tools` across when present, and write one
`{"messages": [...]}` object. Skip the records that should not become training data:

- **Failed calls.** Keep `metadata.status == 200` only.
- **Empty replies.** Skip a response with neither `content` nor `tool_calls`, rather than
  emitting a conversation that ends on the user's turn.
- **The wrong model's answers**, once a student sits in front of the fallback: filter on
  `metadata.source` for the answers the next iteration should learn from.

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

Check the first converted line and the kept count against the download before uploading. The
reply can carry `reasoning` next to `content` when the fallback is a reasoning model; the
conversion keeps `content`, which is what the application used.

## Conversion guidance

- **The system prompt moves to the job description.** `remove_system_prompt_from_traces`
  (default true) strips leading system messages, so the system prompt's content belongs in
  `task_description`, which says the same thing (`../job-description.md`). Variable input →
  user message, model output → assistant message.
- Every trace is a multi-turn conversation rewritten whole. A simple exchange is a two-turn
  conversation.
- `synthgen.validation_max_total_length` applies to processed examples
  (`../configuration.md`); raise it when traces embed documents or schemas.
- Emit clean JSONL: strip control characters and unicode line separators (U+2028/U+2029).

Any task type can train from traces. Traces with retrieved text: `../task-types.md` § Choosing
a task type.
