# Phase 1: Define the Task

| | |
|---|---|
| **Input** | the first Dataset; whatever the user has given (a written spec, files, traces, a description in chat) |
| **Output** | the scope in the run log; a draft `job_description.json` with the fields that define the task |
| **Gate** | the user confirms the task definition |
| **Uses** | `references/task-types.md`, `references/job-description.md`, `references/data-preparation/overview.md` |

## Goal

Understand the task well enough to write it down: what the model does, what it will receive in
production, and what a correct answer is. The written result is the job description, which every
later step reads. The teacher writes data from it, the judge scores with it, and usually the student
is served with it.

Every session gets there differently. Some users hand over a complete specification, some upload
examples and logs, and some explain the problem in conversation. Read everything the user gave
first, then ask only for what is still missing, a few questions at a time. Users do not always know
every answer yet. Record what is unknown as an open question rather than filling it in yourself.

## What this phase must produce

### The scope

Write these into the scope file that the run log header names. Phase 3 checks the test set against them, so state each one
concretely.

| Item | Done when you can state |
|---|---|
| **The job** | what the model is for, who or what sends it inputs, what happens with its answer, and which mistakes are costly (for example a false alarm against a missed case) |
| **The input inventory** | every kind of input production sends: the sources or channels, the formats and structures (plain text, documents, logs, JSON or XML dumps, tables, conversations, tool lists and their lengths), the languages, the length range, and what varies from request to request (for example the system prompt or the tool list). Mark each kind as common, or rare but important |
| **The out-of-scope inputs** | what the model will get that it should not handle, how often, and what it should answer then |
| **The outputs** | the answer format, the set of possible answers (every label, field or call type), and what makes an answer correct |
| **The task type** | `question-answering` or `chat-completion` (`references/task-types.md`). `question-answering` does not support tools: tool calls of any kind need `chat-completion`, with the tools declared in `job_description.json` |
| **The available data** | labelled rows and who labelled them, traces, the production model and how good it is, and a production system prompt the model must keep using, if there is one |

The input inventory is the item most often left incomplete, because the user's examples show the
common inputs, not all of them. Ask directly: "What else does production send?", "Which sources or
formats are there besides these examples?", "Which inputs are rare but matter most?"

When you notice a case a reasonable person could label either way, ask the user how to label it,
with a short example, instead of deciding it yourself.

### The job description

Write a draft of the fields of `job_description.json` that define the task (format and per-task
rules: `references/job-description.md`).

| Field | What goes in it |
|---|---|
| `task_description` | what the task is and how to answer it, as the user understands it. It may be the user's own specification as written, or a draft you write from the conversation and their examples. You do not need to impose a structure on it. Check that it says what to answer for every output and for every kind of input in the inventory, or that the kind is out of scope. Every rule in it comes from the user: their spec, their answers, or their labelled examples. Record the source of each rule in the run log, and ask instead of inventing a rule for a case nobody covered |
| `llm_as_a_judge_instructions` | what a good answer is and what fails, written as the judge's rubric (`references/job-description.md` § Judge score modes). It is part of saying what "correct" means, so it is written here. Phase 2 decides whether the judge is the primary metric or a second signal |
| `tools` | for `chat-completion` with tools: the tool definitions the model may call |

The two other fields, `synthetic_data_generation_instructions` and `trace_processing_instructions`,
are not part of the task definition. They say how to make data: how to generate rows and how to
relabel traces. They are written in the data phases, for each run that makes rows
(`protocols/make-rows.md`): in Phase 3 for each test-set gap, and in Phase 4 for the training rows.

## Exit

Present the task definition to the user in one message:

- the input inventory as a list, each kind marked common or rare;
- the out-of-scope inputs and the outputs;
- the available data;
- the draft job description;
- the open questions.

The user confirms or corrects it. Save the draft in the project directory, write the scope into the
run log header (`protocols/run-log.md` § 1. Header), and mark Phase 1 done. The user agrees the job
description at the end of Phase 2, after the teacher check. Once agreed, it is not changed.
