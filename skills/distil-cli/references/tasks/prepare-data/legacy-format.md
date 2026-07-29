# Legacy Data Format

Prepared train/test datasets used to be flat rows with `question`, `answer`, and (for open book QA) `context` columns. They are now a single `messages` conversation per example, mirroring the chat format the model is trained and served on. This page maps the old format to the new one so you can migrate existing datasets. Datasets in the old format can still be uploaded, but the version is not auto-detected — pass `--data-version 0` to `distil model upload-data` (the `data-version` query parameter on the API). The default is `1`, the `messages` format described here.

> This change applies to **prepared datasets** uploaded with `distil model upload-data`. The raw traces uploaded with `distil traces upload` are unchanged — they stay in the OpenAI chat-completions format (including OpenAI-style tool calls where `arguments` is a stringified JSON blob). See `references/tasks/upload-and-process-traces.md`.

## What Changed

- **One `messages` array per example.** The old `question` and `answer` columns are replaced by a conversation: a `user` turn (the input) and an `assistant` turn (the expected output).
- **Open book QA keeps `context`** as a sibling field alongside `messages`, rather than a third column.
- **Tool calls use the HuggingFace format.** The call lives in the assistant turn's `tool_calls` array, and `arguments` is a real JSON object — not the old stringified `answer` with a `parameters` key, and not OpenAI's stringified `arguments`.

## Question Answering, Classification, Closed Book QA

**Old**

```json
{"question": "What is the total amount due?", "answer": "$540"}
```

**New**

```json
{"messages": [{"role": "user", "content": "What is the total amount due?"}, {"role": "assistant", "content": "$540"}]}
```

## Open Book QA (RAG)

The `context` moves from a column to a sibling field of `messages`.

**Old**

```json
{"question": "How many students enrolled?", "context": "The university enrolled 5,984 students...", "answer": "5,984"}
```

**New**

```json
{"messages": [{"role": "user", "content": "How many students enrolled?"}, {"role": "assistant", "content": "5,984"}], "context": "The university enrolled 5,984 students..."}
```

## Tool Calling

The `answer` string is replaced by an assistant `tool_calls` array. `arguments` is a JSON object (HuggingFace format), not a stringified blob, and there is no more `parameters` key. An assistant turn that makes a tool call omits `content` (an empty string `""` is also accepted).

**Old**

```json
{"question": "What's the weather in New York?", "answer": "{\"name\":\"get_weather\",\"parameters\":{\"location\":\"New York, NY\"}}"}
```

**New**

```json
{"messages": [{"role": "user", "content": "What's the weather in New York?"}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "get_weather", "arguments": {"location": "New York, NY"}}}]}]}
```

## Multi-Turn Tool Calling

The conversation history that used to be a stringified JSON array in `question`, plus the separate `answer` tool call, are now a single `messages` array. The target tool call is the final assistant turn.

**Old**

```json
{"question": "[{\"role\": \"user\", \"content\": \"List files here.\"}, {\"role\": \"assistant\", \"content\": \"\", \"tool_calls\": [{\"type\": \"function\", \"function\": {\"name\": \"ls\", \"arguments\": {}}}]}, {\"role\": \"user\", \"content\": \"Show me config.txt.\"}]", "answer": "{\"name\": \"cat\", \"parameters\": {\"file_name\": \"config.txt\"}}"}
```

**New**

```json
{"messages": [{"role": "user", "content": "List files here."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "ls", "arguments": {}}}]}, {"role": "user", "content": "Show me config.txt."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "cat", "arguments": {"file_name": "config.txt"}}}]}]}
```

In the old multi-turn format the assistant turns *inside* the conversation history already used `arguments` (a JSON object), while the separate `answer` used `parameters` (a stringified object). The new format uses `arguments` consistently everywhere, in one array.

For the full per-task specification, see `overview.md` and the task-specific guides in this directory.
