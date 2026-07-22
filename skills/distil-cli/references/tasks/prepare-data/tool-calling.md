# Tool Calling Data Preparation

Task type: `tool-calling-closed-book`

Use tool calling when the model needs to select and invoke the appropriate function or API based on user requests. The model learns to map natural language queries to structured tool calls with correct parameters.

## Model Compatibility

**Student models:** Qwen3, Qwen3.5, Llama 3-family, LFM2/LFM2.5, FunctionGemma, and Gemma 4 models. **Teachers:** any teacher except `deepseek.r1`, `deepseek.r1-thinking`, `deepseek.v3.1`, `Qwen3-480B-A35B-Coder`, and `Qwen2.5-VL-72B-Instruct` — see `references/model-catalog.md` for the full compatibility matrix and the exact teacher config strings.

## Example Use Cases

- **Voice assistants** -- Map spoken commands to smart home APIs
- **Chatbot actions** -- Convert user intents to CRM/database operations
- **Code generation** -- Transform natural language to API calls
- **Workflow automation** -- Route requests to appropriate microservices
- **Command interfaces** -- Parse user input into system commands

## Data Format

Each example is a `messages` conversation with a `user` turn (the request) and an `assistant` turn whose `tool_calls` array holds the expected call.

| Field | Description |
|-------|-------------|
| `messages` | A `user` turn holding the request and an `assistant` turn holding the tool call in its `tool_calls` array |

**Important:** Tool calls use the **HuggingFace format** — `arguments` is a JSON **object**, not a JSON-encoded string. This differs from OpenAI's chat-completions format, where `arguments` is a stringified JSON blob. The assistant turn that makes a tool call omits `content` (an empty string `""` is also accepted). The tool *schemas* in `job_description.json` still use the OpenAI function format shown below — only the emitted call changes.

## job_description.json

Tool calling requires two fields: `task_description` and `tools`.

```json
{
  "task_description": "Respond with the next tool call to complete the user's request",
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get the current weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "City name, e.g., 'San Francisco, CA'"
            },
            "unit": {
              "type": "string",
              "enum": ["celsius", "fahrenheit"],
              "description": "Temperature unit"
            }
          },
          "required": ["location"],
          "additionalProperties": false
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "send_email",
        "description": "Send an email to a recipient",
        "parameters": {
          "type": "object",
          "properties": {
            "to": {
              "type": "string",
              "description": "Recipient email address"
            },
            "subject": {
              "type": "string",
              "description": "Email subject line"
            },
            "body": {
              "type": "string",
              "description": "Email body content"
            }
          },
          "required": ["to", "subject", "body"],
          "additionalProperties": false
        }
      }
    }
  ]
}
```

**Fields:**
- `task_description`: Describes the main task
- `tools`: List of JSON Schemas describing available tools (follows OpenAI function calling format)

## Train/Test Data Examples

> **Prefer JSONL over CSV for tool calling.** The `messages` array contains a nested tool call — in CSV it must be a single quoted column with doubled quotes, a common source of malformed uploads. JSONL handles the nesting cleanly.

### JSONL format (recommended)

```json
{"messages": [{"role": "user", "content": "What's the weather like in New York?"}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "get_weather", "arguments": {"location": "New York, NY", "unit": "fahrenheit"}}}]}]}
{"messages": [{"role": "user", "content": "Send an email to john@example.com saying the meeting is confirmed"}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "send_email", "arguments": {"to": "john@example.com", "subject": "Meeting Confirmation", "body": "The meeting is confirmed."}}}]}]}
```

### CSV format (works but error-prone)

CSV uses a single `messages` column; each cell holds the same JSON array as the JSONL line above, quoted per CSV rules (double quotes inside the value are doubled).

```csv
messages
"[{""role"": ""user"", ""content"": ""What's the weather like in New York?""}, {""role"": ""assistant"", ""tool_calls"": [{""type"": ""function"", ""function"": {""name"": ""get_weather"", ""arguments"": {""location"": ""New York, NY"", ""unit"": ""fahrenheit""}}}]}]"
```

**Requirements:** Minimum 20 examples. Include examples for all tools.

## config.yaml

```yaml
base:
  task: tool-calling-closed-book
```

**Note:** Tool calling only supports Qwen3, Qwen3.5, Llama 3-family, LFM2/LFM2.5, FunctionGemma, and Gemma 4 student models. See `references/model-catalog.md` for the shortlist.

## Unstructured Data (Optional)

Domain-specific scenarios to guide synthetic data generation. Single column: `context`.

### JSONL format

```json
{"context": "User wants to check weather before their trip to Paris next week."}
{"context": "User needs to send a follow-up email to a client about the proposal."}
{"context": "User is scheduling a meeting with the engineering team for tomorrow."}
```

## Tips

1. **Clear tool descriptions** -- Make function descriptions unambiguous.
2. **Comprehensive parameter descriptions** -- Help the model understand what each parameter expects.
3. **Varied examples** -- Show different ways users might request the same action.
4. **Valid JSON** -- Ensure every `arguments` object is valid JSON and matches the tool schema.
5. **Supported models** -- Qwen3, Qwen3.5, Llama 3-family, LFM2/LFM2.5, FunctionGemma, and Gemma 4 student models. See `references/model-catalog.md`.
6. **Use `arguments`, not `parameters`** -- Tool calls (single-turn and multi-turn) use the HuggingFace shape `{"type": "function", "function": {"name": ..., "arguments": {...}}}` with `arguments` as a JSON object. The old `{"name": ..., "parameters": ...}` stringified form is no longer used.
