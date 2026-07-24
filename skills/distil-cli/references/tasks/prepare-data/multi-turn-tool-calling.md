# Multi-Turn Tool Calling Data Preparation

Task type: `multi-turn-tool-calling-closed-book`

Use multi-turn tool calling when the model needs to generate function calls within a conversational context. Unlike single-turn tool calling where each input is independent, multi-turn tool calling takes a conversation history (user and assistant turns, optionally with tool results) and generates the next appropriate function call based on the full context.

## Model Compatibility

**Students:** Qwen3, Qwen3.5, Llama 3-family, LFM2/LFM2.5, FunctionGemma, and Gemma 4. **Teachers:** any teacher except `deepseek.r1`, `deepseek.r1-thinking`, `deepseek.v3.1`, `Qwen3-480B-A35B-Coder`, and `Qwen2.5-VL-72B-Instruct`.

See `references/model-catalog.md` for the full catalog and the authoritative constraint list.

## Example Use Cases

- **File system assistants** -- Navigate directories and manage files through conversation
- **Database query interfaces** -- Build complex queries through iterative refinement
- **DevOps chatbots** -- Execute sequences of infrastructure commands conversationally
- **Smart home controllers** -- Chain home automation commands naturally
- **Customer service bots** -- Handle multi-step service requests in dialogue
- **IDE assistants** -- Execute code operations through conversational commands

## Data Format

Each example is a single `messages` array holding the whole conversation — the user/assistant turns (optionally with `tool` result turns) *and* the final assistant tool call the model should learn to produce.

| Field | Description |
|-------|-------------|
| `messages` | The full conversation: alternating `user` and `assistant` turns (optionally with `tool` result turns), ending with the target `assistant` tool call |

**Important:**
- The whole conversation, including the target tool call, lives in one `messages` array — there is no separate `answer`.
- Each assistant turn must contain **exactly one function call** (multiple function calls per turn are not supported).
- Tool calls use the **HuggingFace format**: `arguments` is a JSON object, not a stringified JSON blob.

## Conversation Turn Format

Each turn in the conversation is an object with:
- `role`: `"user"`, `"assistant"`, or `"tool"` (a tool-result turn)
- `content`: The text content of the message. Assistant turns that make a tool call omit `content` (an empty string `""` is also accepted).
- `tool_calls`: (assistant only) An array containing exactly one tool call made by the assistant
- `tool_call_id`: (tool only, optional) The `id` of the assistant tool call this result answers. Only needed to disambiguate; it can be omitted when each tool result directly follows the assistant turn that requested it.

`tool` turns are optional — they carry the output of the preceding assistant tool call in `content` and can be followed by either the next `user` turn or the next `assistant` tool call. The first message must be a `user` turn and the last must be the target `assistant` tool call.

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

```json
{"messages": [{"role": "user", "content": "Please list all the files in my current directory."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "ls", "arguments": {}}}]}, {"role": "user", "content": "Navigate to the backup directory."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "cd", "arguments": {"folder": "backup"}}}]}, {"role": "user", "content": "Show me what's inside config.txt."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "cat", "arguments": {"file_name": "config.txt"}}}]}]}
{"messages": [{"role": "user", "content": "Show me all files here including hidden ones."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "ls", "arguments": {"a": true}}}]}, {"role": "user", "content": "Change directory to research."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "cd", "arguments": {"folder": "research"}}}]}, {"role": "user", "content": "Create a file called experiment_log.txt."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "touch", "arguments": {"file_name": "experiment_log.txt"}}}]}]}
```

A conversation may also include `tool` turns carrying the result of the preceding tool call:

```json
{"messages": [{"role": "user", "content": "Show me what's inside notes.txt."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "cat", "arguments": {"file_name": "notes.txt"}}}]}, {"role": "tool", "content": "Buy milk. Call the plumber."}, {"role": "user", "content": "Now delete the file."}, {"role": "assistant", "tool_calls": [{"type": "function", "function": {"name": "rm", "arguments": {"file_name": "notes.txt"}}}]}]}
```

### Understanding the Conversation Structure

Here is an expanded view of what a single `messages` array looks like. The earlier turns are the conversation history; the final assistant turn is the tool call the model is trained to produce:

```json
[
  {
    "role": "user",
    "content": "Please list all the files in my current directory."
  },
  {
    "role": "assistant",
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
  },
  {
    "role": "assistant",
    "tool_calls": [
      {
        "type": "function",
        "function": {
          "name": "cat",
          "arguments": {"file_name": "config.txt"}
        }
      }
    ]
  }
]
```

Given the history through the last user turn, the model learns to produce the final assistant tool call: `{"type": "function", "function": {"name": "cat", "arguments": {"file_name": "config.txt"}}}`.

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

Required teacher: any except `deepseek.r1`, `deepseek.r1-thinking`, `deepseek.v3.1`, `Qwen3-480B-A35B-Coder`, and `Qwen2.5-VL-72B-Instruct`. See `references/model-catalog.md` for the exact config strings.

**If training from traces** (not from a prepared dataset), no extra configuration is needed — the platform processes every trace as a multi-turn conversation and rewrites it as a whole, preserving the conversational context this task type relies on. See `references/tasks/upload-and-process-traces.md`.


## Tips

1. **Clear tool descriptions** -- Make function descriptions unambiguous.
2. **Comprehensive parameter descriptions** -- Help the model understand what each parameter expects.
3. **Varied conversation lengths** -- Include examples with different numbers of turns.
4. **Valid JSON** -- Ensure every `messages` array is valid JSON and each `arguments` object matches the tool schema.
5. **Context-dependent examples** -- Show cases where the next tool call depends on previous conversation context.
6. **Use JSONL** -- The `messages` array is a nested conversation with tool calls, and JSONL handles that nesting cleanly.
7. **Supported models** -- Qwen3, Qwen3.5, Llama 3-family, LFM2/LFM2.5, FunctionGemma, and Gemma 4 students. Any teacher works except `deepseek.r1`, `deepseek.r1-thinking`, `deepseek.v3.1`, `Qwen3-480B-A35B-Coder`, and `Qwen2.5-VL-72B-Instruct`. See `references/model-catalog.md`.
