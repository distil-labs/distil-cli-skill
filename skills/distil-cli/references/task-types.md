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

Job description types: `job-description.md`.

## Deprecated task types

Accepted with a warning; do not use for new work:

| Deprecated | Use instead |
|---|---|
| `information-extraction` | `question-answering` |
| `question-answering-open-book-synthetic-context` | `question-answering-open-book` (synthetic context generation no longer supported) |

## Choosing a task type

| The output is... | Choose |
|---|---|
| Free text, any non-RAG application (answers, extractions, transformations, JSON documents) | `question-answering` |
| One label from a fixed set | `classification` |
| A structured call against a fixed tool schema | `tool-calling-closed-book` |
| The next tool call given a conversation history | `multi-turn-tool-calling-closed-book` |
| RAG only: answer grounded in a retrieved chunk supplied at inference | `question-answering-open-book` |
| Memorize a RAG-like database, no retrieval at inference, answers from memory | `question-answering-closed-book` |

The QA rule:

- Open book is exclusively for RAG: the model expects a retrieved chunk in `context` at
  inference. No retriever in production means it is the wrong choice.
- Closed book is exclusively for memorizing a knowledge database that exists but is not
  retrieved at inference; the knowledge is baked in during training.
- Every non-RAG application uses plain `question-answering`.

Notes:
- `question-answering` is the catch-all for any text-in text-out problem, at the cost of the
  schema validation that classification and tool calling get.
