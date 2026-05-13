# Tool Calling Data Preparation

Task type: `tool-calling-closed-book`

Use tool calling when the model needs to select and invoke the appropriate function or API based on user requests. The model learns to map natural language queries to structured tool calls with correct parameters.

## Model Compatibility

**Student models:** Only Qwen3 and Llama 3-family models. **Teachers:** restricted to the 10-model tool-calling allowlist — see `references/model-catalog.md` for the full compatibility matrix and the exact teacher config strings.

## Example Use Cases

- **Voice assistants** -- Map spoken commands to smart home APIs
- **Chatbot actions** -- Convert user intents to CRM/database operations
- **Code generation** -- Transform natural language to API calls
- **Workflow automation** -- Route requests to appropriate microservices
- **Command interfaces** -- Parse user input into system commands

## Data Columns

| Column | Description |
|--------|-------------|
| `question` | The input containing the user request or current state |
| `answer` | The tool call as a JSON string (with escaped quotes) |

**Important:** The `answer` field must be a JSON **string** (with escaped quotes), not a JSON object.

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

> **Prefer JSONL over CSV for tool calling.** The `answer` field is a JSON string with nested escaped quotes — CSV double-escaping is a common source of malformed uploads. JSONL handles the escaping cleanly.

### JSONL format (recommended)

```json
{"question": "What's the weather like in New York?", "answer": "{\"name\":\"get_weather\",\"parameters\":{\"location\":\"New York, NY\",\"unit\":\"fahrenheit\"}}"}
{"question": "Send an email to john@example.com saying the meeting is confirmed", "answer": "{\"name\":\"send_email\",\"parameters\":{\"to\":\"john@example.com\",\"subject\":\"Meeting Confirmation\",\"body\":\"The meeting is confirmed.\"}}"}
```

### CSV format (works but error-prone)

| question | answer |
|----------|--------|
| What's the weather like in New York? | `"{\"name\":\"get_weather\",\"parameters\":{\"location\":\"New York, NY\",\"unit\":\"fahrenheit\"}}"` |
| Send an email to john@example.com saying the meeting is confirmed | `"{\"name\":\"send_email\",\"parameters\":{\"to\":\"john@example.com\",\"subject\":\"Meeting Confirmation\",\"body\":\"The meeting is confirmed.\"}}"` |

**Requirements:** Minimum 20 examples. Include examples for all tools.

## config.yaml

```yaml
base:
  task: tool-calling-closed-book
```

**Note:** Tool calling only supports Qwen3 and Llama 3-family student models. See `references/model-catalog.md` for the shortlist.

## Unstructured Data (Optional)

Domain-specific scenarios to guide synthetic data generation. Single column: `context`.

### JSONL format

```json
{"context": "User wants to check weather before their trip to Paris next week."}
{"context": "User needs to send a follow-up email to a client about the proposal."}
{"context": "User is scheduling a meeting with the engineering team for tomorrow."}
```

## Using the Trained Model

The model outputs JSON tool calls. Parse and execute:

```python
import json

response = model.generate("What's the weather in Tokyo?")
tool_call = json.loads(response)

if tool_call["name"] == "get_weather":
    result = get_weather(**tool_call["parameters"])
```

## Tips

1. **Clear tool descriptions** -- Make function descriptions unambiguous.
2. **Comprehensive parameter descriptions** -- Help the model understand what each parameter expects.
3. **Varied examples** -- Show different ways users might request the same action.
4. **Valid JSON** -- Ensure all answer fields contain properly escaped JSON strings.
5. **Supported models** -- Only Qwen3 and Llama 3-family student models. See `references/model-catalog.md`.
6. **`parameters` vs `arguments`** -- The training `answer` uses `{"name": ..., "parameters": ...}`. Conversation histories (in multi-turn tool calling) still use OpenAI's `{"function": {"name": ..., "arguments": ...}}`. Don't mix them.
