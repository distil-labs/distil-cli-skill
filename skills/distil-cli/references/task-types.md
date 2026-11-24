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
| Tool calls of any kind, including a single call for a single user message | `chat-completion`, with the tools declared in `job_description.json`; a single-turn call is a two-message row (one user message, one assistant turn with `tool_calls`) |

`question-answering` is the catch-all for any text-in text-out problem, labels included. When
the input carries a document or retrieved text, put it inside the user message. Conversations, with
or without tools and tool results, are `chat-completion`. `chat-completion-agentic` is rejected:
change `base.task` to `chat-completion` and keep the job description, data and config as they are.

**Tool calling is `chat-completion` only, with the tools declared in `job_description.json`.**
Tool definitions written into the text of a `question-answering` row (for example as JSON in a
per-row system message, with the call as a JSON answer) are not tools to the platform: it runs
the task as ordinary text. Synthetic generation then has the teacher write new tool definitions
as text for every row, and nothing checks them; on a BFCL-style task about half came out as
invalid JSON. Two setups work: `chat-completion` with the tools declared, or
`question-answering` without strict tool calls, where the answer picks an option from a list given
in the input, as a label.

Data formats and rules: `data-preparation/question-answering.md`, and
`data-preparation/chat-completion.md` for chat completion.
