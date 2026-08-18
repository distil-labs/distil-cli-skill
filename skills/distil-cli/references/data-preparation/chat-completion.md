# Data Preparation: Chat Completion (both variants)

Task values: `chat-completion` and `chat-completion-agentic`. Rows are whole conversations in
one `messages` array. They must start with a user message and end with an assistant message.
Assistant turns carry `content`, `tool_calls`, or both (at least one; parallel calls allowed).
This is the difference from multi-turn tool calling, which forces exactly one call and empty
content per assistant turn.

The two variants differ on one axis: tool results.

- `chat-completion`: user and assistant roles only. An assistant turn is always followed by the
  next user turn. `role: "tool"` messages are never valid; there is no dedicated error for
  them, they fail as an invalid `assistant -> tool` (or `user -> tool`) transition. If the data
  has tool results, the task is `chat-completion-agentic`.
- `chat-completion-agentic`: `role: "tool"` results are admitted, but only after an assistant
  turn that made tool calls. After a tool result: another `tool` result, an `assistant` turn,
  or the next `user` turn. Conversations still end with an assistant message, typically the
  final text answer.

`chat-completion`, a content turn, a content+call turn, and a call-only turn:

```jsonl
{"messages": [{"role": "user", "content": "Hi, can you help me plan a trip?"}, {"role": "assistant", "content": "Of course. Where to?"}, {"role": "user", "content": "Paris. What's the weather?"}, {"role": "assistant", "content": "Let me check.", "tool_calls": [{"id": "call_1", "type": "function", "function": {"name": "get_weather", "arguments": {"location": "Paris"}}}]}]}
{"messages": [{"role": "user", "content": "Search for events in Berlin."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "search", "arguments": {"query": "events in Berlin"}}}]}]}
```

`chat-completion-agentic`, a full loop with a chained call:

```jsonl
{"messages": [{"role": "user", "content": "Look up the tallest building, then tell me the weather there."}, {"role": "assistant", "tool_calls": [{"id": "call_1", "type": "function", "function": {"name": "search", "arguments": {"query": "tallest building"}}}]}, {"role": "tool", "tool_call_id": "call_1", "content": "Burj Khalifa, Dubai"}, {"role": "assistant", "tool_calls": [{"id": "call_2", "type": "function", "function": {"name": "get_weather", "arguments": {"location": "Dubai"}}}]}, {"role": "tool", "tool_call_id": "call_2", "content": "34C, sunny"}, {"role": "assistant", "content": "The Burj Khalifa in Dubai, where it's 34C and sunny."}]}
```

Conversation rules (validated at parse):

- Roles: user / assistant (+ tool for agentic only). NO system messages; `task_description`
  becomes the generated system prompt. Strict alternation otherwise: user → assistant,
  assistant → user.
- Assistant turns: `content`, `tool_calls`, or both; a turn with neither fails.
  `function.arguments` is a JSON object, not a string.
- `tools` in job_description.json: optional for `chat-completion` (a job with no tools and tool
  calls in the data fails with a dedicated error), REQUIRED for `chat-completion-agentic`.
  Every call validates against the schemas.
- `tool_call_id` on results must match the calls of the preceding assistant turn. A single
  call/result pair is fixed up automatically; parallel calls need matching ids. Results are
  optional per call (fewer results than calls is fine, more fails).
- Neither variant supports `visual_task` or a `context` column.

Synthetic generation caps tool calls per generated assistant turn at
`synthgen.max_tool_calls_per_turn` (default: 1 with tools declared, 0 without). Set an integer
or `"unlimited"` for parallel calls; it must stay consistent with whether tools exist
(`../configuration.md` § Cross-field validation).

Turn expansion: conversations are automatically split into one example per assistant turn, each
predicted from the full prefix (tool results included), for training and evaluation both. So
the model learns intermediate agentic calls AND final answers, and eval line counts exceed
uploaded row counts. Pre-split data is detected (prefix overlap) and passed through unchanged.

Evaluation uses the tool-calling metric set plus the LLM judge (`../evaluation-metrics.md`).
Parallel calls are compared positionally, so reference call order matters. Turns with no calls
on either side score 1 on the tool metrics; the judge carries the text quality signal.

Model constraints: with tools declared (always, for agentic), both student and teacher must be
tool-calling capable (`../model-catalog.md`). A tool-free `chat-completion` job has no
restriction.
