# Data Preparation: Chat Completion

Task value: `chat-completion`. It covers a tool-free chat model, a model whose calls your code
runs without feeding results back, and an agent loop that reads tool results. A config with
`chat-completion-agentic` is rejected: change `base.task` to `chat-completion`, nothing else.

Rows are whole conversations in one `messages` array. They must start with a user message and
end with an assistant message. Assistant turns carry `content`, `tool_calls`, or both (at least
one; parallel calls allowed).

`role: "tool"` results are admitted only after an assistant turn that made tool calls. After a
tool result: another `tool` result, an `assistant` turn, or the next `user` turn. A tool result
anywhere else has no dedicated error: it fails as an invalid `user -> tool` (or `assistant ->
tool`) transition. A job without tools has no calls, so its rows have no tool results.

A content turn, a content+call turn whose result is not fed back, and a call-only turn:

```jsonl
{"messages": [{"role": "user", "content": "Hi, can you help me plan a trip?"}, {"role": "assistant", "content": "Of course. Where to?"}, {"role": "user", "content": "Paris. What's the weather?"}, {"role": "assistant", "content": "Let me check.", "tool_calls": [{"id": "call_1", "type": "function", "function": {"name": "get_weather", "arguments": {"location": "Paris"}}}]}]}
{"messages": [{"role": "user", "content": "Search for events in Berlin."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "search", "arguments": {"query": "events in Berlin"}}}]}]}
```

A full loop with a chained call, the results fed back:

```jsonl
{"messages": [{"role": "user", "content": "Look up the tallest building, then tell me the weather there."}, {"role": "assistant", "tool_calls": [{"id": "call_1", "type": "function", "function": {"name": "search", "arguments": {"query": "tallest building"}}}]}, {"role": "tool", "tool_call_id": "call_1", "content": "Burj Khalifa, Dubai"}, {"role": "assistant", "tool_calls": [{"id": "call_2", "type": "function", "function": {"name": "get_weather", "arguments": {"location": "Dubai"}}}]}, {"role": "tool", "tool_call_id": "call_2", "content": "34C, sunny"}, {"role": "assistant", "content": "The Burj Khalifa in Dubai, where it's 34C and sunny."}]}
```

Conversation rules (validated at parse):

- Roles: user / assistant (+ tool, after an assistant turn with calls). NO system messages; `task_description`
  becomes the generated system prompt. With `base.variable_system_prompt` on, one leading
  system message is allowed (`../variable-system-prompts.md`). Strict alternation otherwise: user → assistant,
  assistant → user.
- Assistant turns: `content`, `tool_calls`, or both; a turn with neither fails.
  `function.arguments` is a JSON object, not a string.
- `tools` in job_description.json: optional. Every call validates against the schemas, and tool calls or tool
  results in the data with no tools declared fail the create.
- `tool_call_id` on results must match the calls of the preceding assistant turn. A single
  call/result pair is fixed up automatically; parallel calls need matching ids. Results are
  optional per call (fewer results than calls is fine, more fails).
- Chat completion does not support `visual_task`.

## Synthetic data with tools

Synthgen does not target tools one at a time: every generation call sees all the declared
tools, so the teacher decides which tool each example uses, and some tools can end up rare or
missing. When the job description declares tools, add a mutator with one value per tool, so
every generation call is asked for a specific tool:

```yaml
synthgen:
  mutators:
    - name: tool
      values:
        - "the conversation calls get_weather"
        - "the conversation calls search"
        - "the conversation calls book_table"
```

Also add a value such as "the assistant answers in text with no tool
call" when the model should sometimes answer without a tool. Set `target_distribution:
match_seed` to follow the tool mix of the split's rows instead of an even split, or give one
weight per value (`../mutators.md`). Check the tool counts in the smoke output (the
synthetic-data-generation stage, Step 4, axis 3).

## Tool calls per turn

Synthetic generation caps tool calls per generated assistant turn at
`synthgen.max_tool_calls_per_turn` (default: 1 with tools declared, 0 without). Set an integer
or `"unlimited"` for parallel calls; it must stay consistent with whether tools exist
(`../configuration.md` § Cross-field validation).

Turn expansion: by default a conversation becomes one example per assistant turn, each
predicted from the full prefix (tool results included), for training and evaluation. The model
learns the intermediate calls and the final answers, and evaluation counts more lines than
uploaded rows. Settings: `../configuration.md` § Conversation expansion.

Evaluation uses the conversation metric set, which includes the LLM judge
(`../evaluation-metrics.md`).
Parallel calls are compared positionally, so reference call order matters. Turns with no calls
on either side score 1 on the tool metrics; the judge carries the text quality signal.

Model constraints with tools declared: `../model-catalog.md` § Tool-calling compatibility.
