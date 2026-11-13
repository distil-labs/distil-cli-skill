# Task Types

`base.task` in `config.yaml` selects the task.

## Task types

| Task | `base.task` value | Job description type |
|---|---|---|
| Question Answering | `question-answering` | QA |
| Chat Completion | `chat-completion` | Conversation |

Job description types: `job-description.md`.

## Choosing a task type

| The output is... | Choose |
|---|---|
| Free text for a single input (answers, extractions, transformations, JSON documents) | `question-answering` |
| One label from a fixed set | `question-answering` with a mutator over the labels (`data-preparation/question-answering.md` § Classifying with question-answering) |
| A conversational turn: free text, tool calls, or both, with or without the tool results fed back (an agentic loop) | `chat-completion` |

`question-answering` is the catch-all for any text-in text-out problem, labels included. When
the input carries a document or retrieved text, put it inside the user message. Conversations, with
or without tools and tool results, are `chat-completion`. `chat-completion-agentic` is rejected:
change `base.task` to `chat-completion` and keep the job description, data and config as they are.

Data formats and rules: `data-preparation/question-answering.md`, and
`data-preparation/chat-completion.md` for chat completion.
