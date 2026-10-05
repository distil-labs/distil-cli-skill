# Task Types

`base.task` in `config.yaml` selects the task.

## Task types

| Task | `base.task` value | Job description type |
|---|---|---|
| Question Answering | `question-answering` | QA |
| Classification | `classification` | Classification |
| Chat Completion | `chat-completion` | Conversation |
| Agentic Chat Completion | `chat-completion-agentic` | Conversation |

Job description types: `job-description.md`.

## Choosing a task type

| The output is... | Choose |
|---|---|
| Free text for a single input (answers, extractions, transformations, JSON documents) | `question-answering` |
| One label from a fixed set | `classification` |
| A conversational turn: free text, tool calls, or both; the model never sees tool results | `chat-completion` |
| An agentic loop: tool calls whose results feed back, ending in a grounded answer | `chat-completion-agentic` |

`question-answering` is the catch-all for any text-in text-out problem, without the label
validation classification gets. When the input carries a document or retrieved text, put it
inside the user message. Data with tool results in it is `chat-completion-agentic`.

Data formats and rules: `data-preparation/question-answering.md`, `data-preparation/classification.md`,
and `data-preparation/chat-completion.md` for both chat completion tasks.
