# Task Types

`base.task` in `config.yaml` selects the task.

## Active task types

| Task | `base.task` value | Context column | Needs unstructured data | Job description type |
|---|---|---|---|---|
| Question Answering | `question-answering` | No | No | QA |
| Classification | `classification` | No | No | Classification |
| Open Book QA (RAG) | `question-answering-open-book` | Yes | Yes | QA |
| Closed Book QA | `question-answering-closed-book` | No | Yes | QA |
| Tool Calling | `tool-calling-closed-book` | No | No | Tool Calling |
| Multi-Turn Tool Calling | `multi-turn-tool-calling-closed-book` | No | No | Tool Calling |
| Chat Completion | `chat-completion` | No | No | Conversation |
| Agentic Chat Completion | `chat-completion-agentic` | No | No | Conversation |

Job description types: `job-description.md`.

## Deprecated task types

Accepted with a warning. Do not use them for new work:

| Deprecated | Use instead |
|---|---|
| `information-extraction` | `question-answering` |
| `question-answering-open-book-synthetic-context` | `question-answering-open-book` (there is no synthetic context generation) |

## Choosing a task type

| The output is... | Choose |
|---|---|
| Free text, any non-RAG application (answers, extractions, transformations, JSON documents) | `question-answering` |
| One label from a fixed set | `classification` |
| A structured call against a fixed tool schema | `tool-calling-closed-book` |
| The next tool call given a conversation history (calls only, no free text) | `multi-turn-tool-calling-closed-book` |
| A conversational turn: free text, tool calls, or both; the model never sees tool results | `chat-completion` |
| An agentic loop: tool calls whose results feed back, ending in a grounded answer | `chat-completion-agentic` |
| RAG only: answer grounded in a retrieved chunk supplied at inference | `question-answering-open-book` |
| Memorize a RAG-like database, no retrieval at inference, answers from memory | `question-answering-closed-book` |

The QA rule:

- Open book is exclusively for RAG: the model expects a retrieved chunk in `context` at
  inference. No retriever in production means it is the wrong choice.
- Closed book is exclusively for memorizing a knowledge database that exists but is not
  retrieved at inference. The knowledge is baked in during training.
- Every non-RAG application uses plain `question-answering`.

`question-answering` is the catch-all for any text-in text-out problem, at the cost of the
schema validation that classification and tool calling get.

The conversation rule:

- Both chat completion tasks allow assistant turns with content, tool calls, or both, and
  parallel tool calls. Multi-turn tool calling forces every assistant turn to be exactly one
  call with empty content.
- `chat-completion` never admits `tool` (result) messages: assistant → user only. Data with
  tool results in it needs `chat-completion-agentic`, where a tool result may follow an
  assistant turn that made calls.
- `tools` in the job description: optional for `chat-completion` (omit for a pure chat model),
  required for `chat-completion-agentic`.
- Neither supports `visual_task`, `context` columns, or `system` messages (the
  `task_description` becomes the system prompt).

Data format for both: `data-preparation/chat-completion.md`.
