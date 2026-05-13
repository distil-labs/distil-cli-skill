# Multi-Turn Tool Calling Data Preparation

Task type: `multi-turn-tool-calling-closed-book`

Use multi-turn tool calling when the model needs to generate function calls within a conversational context. Unlike single-turn tool calling where each input is independent, multi-turn tool calling takes a conversation history (alternating user and assistant messages) and generates the next appropriate function call based on the full context.

## Model Compatibility

**Students:** Only Qwen3 and Llama 3-family. **Teachers:** restricted to a 10-model allowlist — `openai.gpt-oss-120b`, `openai.gpt-oss-120b-thinking`, `Llama-3.1-405B-Instruct`, `Qwen3-235B-A22B-Instruct-2507`, `deepseek.v3.2`, `deepseek.v3.2-thinking`, `zai.glm-5`, `zai.glm-5-thinking`, `moonshotai.kimi-k2-thinking`, `minimax.minimax-m2-thinking`. No other teachers work for this task type.

See `references/model-catalog.md` for the full catalog and the authoritative constraint list.

## Example Use Cases

- **File system assistants** -- Navigate directories and manage files through conversation
- **Database query interfaces** -- Build complex queries through iterative refinement
- **DevOps chatbots** -- Execute sequences of infrastructure commands conversationally
- **Smart home controllers** -- Chain home automation commands naturally
- **Customer service bots** -- Handle multi-step service requests in dialogue
- **IDE assistants** -- Execute code operations through conversational commands

## Data Columns

| Column | Description |
|--------|-------------|
| `question` | A JSON array (as a string) representing the conversation history with alternating user and assistant messages |
| `answer` | The tool call to be invoked according to the provided schema |

**Important:**
- The `question` field contains a **stringified JSON array** representing the conversation
- Each assistant turn must contain **exactly one function call** (multiple function calls per turn are not supported)
- The `answer` field must be a JSON string (with escaped quotes), not a JSON object

## Conversation Turn Format

Each turn in the conversation is an object with:
- `role`: Either `"user"` or `"assistant"`
- `content`: The text content of the message (empty string for assistant turns with tool calls)
- `tool_calls`: (assistant only) An array containing exactly one tool call made by the assistant

## job_description.json

Multi-turn tool calling requires two fields: `task_description` and `tools`.

```json
{
  "task_description": "You are an intelligent AI assistant that helps users navigate and manage files in a file system. Given a command or request from the user, call the appropriate file system function to complete the request.",
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "ls",
        "description": "List the contents of the current directory.",
        "parameters": {
          "type": "object",
          "properties": {
            "a": {
              "type": "boolean",
              "description": "Show hidden files and directories. Defaults to False.",
              "default": false
            }
          },
          "required": []
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "cd",
        "description": "Change the current working directory to the specified folder.",
        "parameters": {
          "type": "object",
          "properties": {
            "folder": {
              "type": "string",
              "description": "The folder of the directory to change to."
            }
          },
          "required": ["folder"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "cat",
        "description": "Display the contents of a file from the current directory.",
        "parameters": {
          "type": "object",
          "properties": {
            "file_name": {
              "type": "string",
              "description": "The name of the file to display."
            }
          },
          "required": ["file_name"]
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

### JSONL format

```json
{"question": "[{\"role\": \"user\", \"content\": \"Please list all the files in my current directory.\"}, {\"role\": \"assistant\", \"content\": \"\", \"tool_calls\": [{\"type\": \"function\", \"function\": {\"name\": \"ls\", \"arguments\": {}}}]}, {\"role\": \"user\", \"content\": \"Navigate to the backup directory.\"}, {\"role\": \"assistant\", \"content\": \"\", \"tool_calls\": [{\"type\": \"function\", \"function\": {\"name\": \"cd\", \"arguments\": {\"folder\": \"backup\"}}}]}, {\"role\": \"user\", \"content\": \"Show me what's inside config.txt.\"}]", "answer": "{\"name\": \"cat\", \"parameters\": {\"file_name\": \"config.txt\"}}"}
{"question": "[{\"role\": \"user\", \"content\": \"Show me all files here including hidden ones.\"}, {\"role\": \"assistant\", \"content\": \"\", \"tool_calls\": [{\"type\": \"function\", \"function\": {\"name\": \"ls\", \"arguments\": {\"a\": true}}}]}, {\"role\": \"user\", \"content\": \"Change directory to research.\"}, {\"role\": \"assistant\", \"content\": \"\", \"tool_calls\": [{\"type\": \"function\", \"function\": {\"name\": \"cd\", \"arguments\": {\"folder\": \"research\"}}}]}, {\"role\": \"user\", \"content\": \"Create a file called experiment_log.txt.\"}]", "answer": "{\"name\": \"touch\", \"parameters\": {\"file_name\": \"experiment_log.txt\"}}"}
```

### Understanding the Conversation Structure

Here is an expanded view of what a single conversation in the `question` field looks like when parsed from JSON:

```json
[
  {
    "role": "user",
    "content": "Please list all the files in my current directory."
  },
  {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "type": "function",
        "function": {
          "name": "ls",
          "arguments": {}
        }
      }
    ]
  },
  {
    "role": "user",
    "content": "Navigate to the backup directory."
  },
  {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "type": "function",
        "function": {
          "name": "cd",
          "arguments": {"folder": "backup"}
        }
      }
    ]
  },
  {
    "role": "user",
    "content": "Show me what's inside config.txt."
  }
]
```

The model receives this conversation history and should output: `{"name": "cat", "parameters": {"file_name": "config.txt"}}`

**Requirements:** Minimum 20 examples. Include examples for all tools.

## Key Differences from Single-Turn Tool Calling

| Aspect | Single-Turn Tool Calling | Multi-Turn Tool Calling |
|--------|--------------------------|-------------------------|
| Input format | Plain text question | JSON array of conversation turns |
| Context | Each request is independent | Model maintains context across turns |
| Use case | One-shot function invocation | Conversational command sequences |
| Training goal | Map query to function | Understand conversation to next function |

## config.yaml

You must specify the task and a compatible teacher model:

```yaml
base:
  task: multi-turn-tool-calling-closed-book
  teacher_model_name: openai.gpt-oss-120b
```

Required teacher: one of the 10-model allowlist. See `references/model-catalog.md` for the exact config strings.

**If training from traces** (not from a prepared dataset), also set `trace_processing.convert_to_single_turn: false`. The default (`true`) splits multi-turn conversations into isolated single-turn examples, which destroys the conversational context this task type relies on. See `references/tasks/upload-and-process-traces.md`.

## Using the Trained Model

The model outputs JSON tool calls. Parse and execute:

```python
import json

# Build conversation history
conversation = [
    {"role": "user", "content": "List files here"},
    {"role": "assistant", "content": "", "tool_calls": [{"type": "function", "function": {"name": "ls", "arguments": {}}}]},
    {"role": "user", "content": "Go to the documents folder"}
]

response = model.generate(json.dumps(conversation))
tool_call = json.loads(response)

if tool_call["name"] == "cd":
    result = cd(**tool_call["parameters"])
```

## Tips

1. **Clear tool descriptions** -- Make function descriptions unambiguous.
2. **Comprehensive parameter descriptions** -- Help the model understand what each parameter expects.
3. **Varied conversation lengths** -- Include examples with different numbers of turns.
4. **Valid JSON** -- Ensure all question fields contain properly escaped JSON arrays and answer fields contain properly escaped JSON strings.
5. **Context-dependent examples** -- Show cases where the next tool call depends on previous conversation context.
6. **Prefer JSONL over CSV** -- The `question` field is a stringified JSON array and the `answer` is a stringified JSON object. CSV double-escaping is a common source of malformed uploads.
7. **Supported models** -- Only Qwen3 and Llama 3-family students. Teachers must be one of the 10-model allowlist. See `references/model-catalog.md`.
