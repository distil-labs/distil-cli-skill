# Tool Calling Data Preparation

Use tool calling when the model needs to select and invoke the appropriate function or API based on user requests. The model learns to map natural language queries to structured tool calls with correct parameters.

**Note:** Only Llama3 family of student models is supported for tool calling.

**Example use cases:**
- Voice assistants — Map spoken commands to smart home APIs
- Chatbot actions — Convert user intents to CRM/database operations
- Code generation — Transform natural language to API calls
- Workflow automation — Route requests to appropriate microservices
- Command interfaces — Parse user input into system commands

## Required Files

### 1. job_description.json

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

### 2. train.csv (or train.jsonl)

| Column | Description |
|--------|-------------|
| `question` | The input containing the user request or current state |
| `answer` | The tool call as a JSON string |

**Important:** The `answer` field must be a JSON string (with escaped quotes), not a JSON object.

**JSONL format:**
```json
{"question": "What's the weather like in New York?", "answer": "{\"name\":\"get_weather\",\"parameters\":{\"location\":\"New York, NY\",\"unit\":\"fahrenheit\"}}"}
{"question": "Send an email to john@example.com saying the meeting is confirmed", "answer": "{\"name\":\"send_email\",\"parameters\":{\"to\":\"john@example.com\",\"subject\":\"Meeting Confirmation\",\"body\":\"The meeting is confirmed.\"}}"}
```

**CSV format:**

| question | answer |
|----------|--------|
| What's the weather like in New York? | `"{\"name\":\"get_weather\",\"parameters\":{\"location\":\"New York, NY\",\"unit\":\"fahrenheit\"}}"` |
| Send an email to john@example.com saying the meeting is confirmed | `"{\"name\":\"send_email\",\"parameters\":{\"to\":\"john@example.com\",\"subject\":\"Meeting Confirmation\",\"body\":\"The meeting is confirmed.\"}}"` |

**Requirements:** Minimum 20 examples. Include examples for all tools.

### 3. test.csv (or test.jsonl)

Same format as train data. Used for evaluation, not training.

### 4. config.yaml

The default configuration works well for most cases. You only need to specify the task:

```yaml
base:
  task: tool-calling-closed-book
```

**Note:** Tool calling only supports Llama3 family student models.

For advanced options (model selection, training parameters, etc.), see `config.md`.

### 5. unstructured.csv (Optional)

Domain-specific scenarios to guide synthetic data generation. Single column: `context`

**JSONL format:**
```json
{"context": "User wants to check weather before their trip to Paris next week."}
{"context": "User needs to send a follow-up email to a client about the proposal."}
{"context": "User is scheduling a meeting with the engineering team for tomorrow."}
```

## Upload and Train

```bash
distil model upload-data <model-id> --data ./my-tool-calling-data
distil model run-teacher-evaluation <model-id>
distil model run-training <model-id>
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

1. **Clear tool descriptions** — Make function descriptions unambiguous
2. **Comprehensive parameter descriptions** — Help the model understand what each parameter expects
3. **Varied examples** — Show different ways users might request the same action
4. **Valid JSON** — Ensure all answer fields contain properly escaped JSON strings
5. **Llama3 only** — Remember only Llama3 family models are supported
