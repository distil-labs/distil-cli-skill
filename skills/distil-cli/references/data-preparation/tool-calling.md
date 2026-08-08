# Data Preparation: Tool Calling

Task value: `tool-calling-closed-book`. Rows are two-message conversations: user = the
request, assistant = the tool call. The assistant message has EMPTY `content` and a
`tool_calls` list; the function's `arguments` is a JSON object (not a string).

```jsonl
{"messages": [{"role": "user", "content": "What is 2 + 2?"}, {"role": "assistant", "content": "", "tool_calls": [{"type": "function", "function": {"name": "calculator", "arguments": {"expression": "2+2"}}}]}]}
```

Rules beyond the shared checklist (`overview.md`):

- Every tool call is validated against the `tools` schemas in job_description.json
  (fields: `../job-description.md`); unknown tool names or schema-invalid arguments fail.
- One tool call per assistant message.
- Student and teacher must support tool calling; see `../model-catalog.md` before picking
  models.
