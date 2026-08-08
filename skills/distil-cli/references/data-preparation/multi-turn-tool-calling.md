# Data Preparation: Multi-Turn Tool Calling

Task value: `multi-turn-tool-calling-closed-book`. Rows are whole conversations in one
`messages` array. They must start with a user message and end with an assistant tool call.
In between, user, assistant-tool-call, and tool-result messages alternate in a valid order.

```jsonl
{"messages": [{"role": "user", "content": "What is 7 * 2?"}, {"role": "assistant", "content": "", "tool_calls": [{"id": "call_1", "type": "function", "function": {"name": "calculator", "arguments": {"expression": "7*2"}}}]}, {"role": "tool", "content": "14", "tool_call_id": "call_1"}, {"role": "user", "content": "What is 2 + 2?"}, {"role": "assistant", "content": "", "tool_calls": [{"type": "function", "function": {"name": "calculator", "arguments": {"expression": "2+2"}}}]}]}
```

Conversation rules (validated at parse):

- Assistant tool-call messages: EMPTY `content`, `tool_calls` with `function.arguments` as a
  JSON object. One tool call per assistant message.
- Tool-result messages: `{"role": "tool", "content": ..., "tool_call_id": ...}` matching the
  preceding assistant call's `id` (a single call/response pair is fixed up automatically).
- Role sequence is validated. Invalid transitions fail with a per-row error.
- Every tool call must validate against the `tools` schemas in job_description.json
  (fields: `../job-description.md`).

Evaluation note: multi-turn conversations are automatically expanded into one eval line
per tool call, so (UTUTU, T) becomes (U,T), (UTU,T), (UTUTU,T). Data already in this
split form is detected (via prefix overlap between examples) and passed through unchanged.

Model constraints: see `../model-catalog.md`. Multi-turn additionally restricts the teacher.
