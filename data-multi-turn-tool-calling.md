# Multi-Turn Tool Calling Data Preparation

Use multi-turn tool calling when the model needs to generate function calls within a conversational context. Unlike single-turn tool calling where each input is independent, multi-turn tool calling takes a conversation history (alternating user and assistant messages) and generates the next appropriate function call based on the full context.

## Model Compatibility

**Student models**: Only Qwen3 and Llama 3-family models are supported for multi-turn tool calling.

**Teacher models**: Multi-turn tool calling requires one of the following teacher models:
- `Qwen3-235B-A22B-Instruct-2507`
- `Llama-3.1-405B-Instruct`
- `openai.gpt-oss-120b`

**Example use cases:**
- File system assistants — Navigate directories and manage files through conversation
- Database query interfaces — Build complex queries through iterative refinement
- DevOps chatbots — Execute sequences of infrastructure commands conversationally
- Smart home controllers — Chain home automation commands naturally
- Customer service bots — Handle multi-step service requests in dialogue
- IDE assistants — Execute code operations through conversational commands

## Required Files

### 1. job_description.json

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

### 2. train.csv (or train.jsonl)

| Column | Description |
|--------|-------------|
| `question` | A JSON array representing the conversation history with alternating user and assistant messages |
| `answer` | The tool call to be invoked according to the provided schema |

**Important:**
- The `question` field contains a **stringified JSON array** representing the conversation
- Each assistant turn must contain **exactly one function call** (multiple function calls per turn are not supported)
- The `answer` field must be a JSON string (with escaped quotes), not a JSON object

### Conversation Format

Each turn in the conversation is an object with:
- `role`: Either `"user"` or `"assistant"`
- `content`: The text content of the message (empty string for assistant turns with tool calls)
- `tool_calls`: (assistant only) An array containing exactly one tool call made by the assistant

**JSONL format:**
```json
{"question": "[{\"role\": \"user\", \"content\": \"Please list all the files in my current directory.\"}, {\"role\": \"assistant\", \"content\": \"\", \"tool_calls\": [{\"type\": \"function\", \"function\": {\"name\": \"ls\", \"arguments\": {}}}]}, {\"role\": \"user\", \"content\": \"Navigate to the backup directory.\"}, {\"role\": \"assistant\", \"content\": \"\", \"tool_calls\": [{\"type\": \"function\", \"function\": {\"name\": \"cd\", \"arguments\": {\"folder\": \"backup\"}}}]}, {\"role\": \"user\", \"content\": \"Show me what's inside config.txt.\"}]", "answer": "{\"name\": \"cat\", \"parameters\": {\"file_name\": \"config.txt\"}}"}
{"question": "[{\"role\": \"user\", \"content\": \"Show me all files here including hidden ones.\"}, {\"role\": \"assistant\", \"content\": \"\", \"tool_calls\": [{\"type\": \"function\", \"function\": {\"name\": \"ls\", \"arguments\": {\"a\": true}}}]}, {\"role\": \"user\", \"content\": \"Change directory to research.\"}, {\"role\": \"assistant\", \"content\": \"\", \"tool_calls\": [{\"type\": \"function\", \"function\": {\"name\": \"cd\", \"arguments\": {\"folder\": \"research\"}}}]}, {\"role\": \"user\", \"content\": \"Create a file called experiment_log.txt.\"}]", "answer": "{\"name\": \"touch\", \"parameters\": {\"file_name\": \"experiment_log.txt\"}}"}
```

### Understanding the Conversation Structure

Here's an expanded view of what a single conversation looks like:

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

### Key Differences from Single-Turn Tool Calling

| Aspect | Single-Turn Tool Calling | Multi-Turn Tool Calling |
|--------|--------------------------|-------------------------|
| Input format | Plain text question | JSON array of conversation turns |
| Context | Each request is independent | Model maintains context across turns |
| Use case | One-shot function invocation | Conversational command sequences |
| Training goal | Map query to function | Understand conversation to next function |

### 3. test.csv (or test.jsonl)

Same format as train data. Used for evaluation, not training.

### 4. config.yaml

The default configuration works well for most cases. You must specify the task and a compatible teacher model:

```yaml
base:
  task: multi-turn-tool-calling-closed-book
  teacher_model_name: openai.gpt-oss-120b
```

**Important:** Multi-turn tool calling requires one of these teacher models:
- `Qwen3-235B-A22B-Instruct-2507`
- `Llama-3.1-405B-Instruct`
- `openai.gpt-oss-120b`

For advanced options (model selection, training parameters, etc.), see `config.md`.

## Upload and Train

```bash
distil model upload-data <model-id> --data ./my-multi-turn-tool-calling-data
distil model run-teacher-evaluation <model-id>
distil model run-training <model-id>
```

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

1. **Clear tool descriptions** — Make function descriptions unambiguous
2. **Comprehensive parameter descriptions** — Help the model understand what each parameter expects
3. **Varied conversation lengths** — Include examples with different numbers of turns
4. **Valid JSON** — Ensure all question fields contain properly escaped JSON arrays and answer fields contain properly escaped JSON strings
5. **Context-dependent examples** — Show cases where the next tool call depends on previous conversation context
6. **Supported models** — Remember only Qwen3 and Llama 3-family student models are supported
7. **Teacher model** — You must use one of the supported teacher models: `Qwen3-235B-A22B-Instruct-2507`, `Llama-3.1-405B-Instruct`, or `openai.gpt-oss-120b`
