# Variable System Prompts

Everything about per-row system prompts. By default a model has one system prompt, built from
the job description and used in every training row, evaluation row and request. One config
key changes that: `base.variable_system_prompt` (default `false`). With it on, every row keeps
its own system message, synthetic data generation writes a different system message for every
example, and the trained model follows the system message each request sends. It is allowed for
every task type.

## When to propose it

- When one model should serve several applications, customers or policies whose instructions
  differ: a rule that changes the answer, a default value, a forbidden option, a term to
  resolve, a different output format.
- When the user's traces carry different system prompts for the same task.
- Not when the application always sends the same system prompt: the fixed prompt is simpler
  and the default.

## The fixed system prompt

With the flag off, every row and request uses the job description's system prompt:

| Task | System prompt |
|---|---|
| `question-answering`, `chat-completion`, `chat-completion-agentic` | `task_description` |

With the flag on, a row without a system message gets this same prompt.

## Where to set it

`base.variable_system_prompt: true` in the `config.yaml` of the uploaded Dataset, before the
first expand, so every stage reads it:

| Stage | Flag on | Flag off |
|---|---|---|
| Loading train and test rows | A row keeps its system message; a row without one gets the fixed prompt | Every row gets the fixed prompt; system messages in the files are replaced |
| Relabelling traces | A trace keeps its system message | A trace gets the fixed prompt |
| Synthetic data generation | The teacher writes a system message for every new example; an example without one is dropped by validation | Every new example gets the fixed prompt |
| Teacher evaluation | The teacher answers each test row with that row's system message; the judge sees it in the prefix | The teacher answers with the fixed prompt |
| Training | The student trains on each row's system message | The student trains on the fixed prompt |

`trace_processing.remove_system_prompt_from_traces` is deprecated and has no effect; this key
decides.

## Data format

- One system message as the first message of a row, exactly as the application will send it:
  ```json
  {"messages": [{"role": "system", "content": "Do not redact person names; agents need the caller's name to verify the account."}, {"role": "user", "content": "..."}, {"role": "assistant", "content": "..."}]}
  ```
- Rows without one are allowed and get the fixed prompt.
- Test rows carry the system messages the application will send, so the test set measures
  whether the model follows them.
- Traces in `openai_messages` format keep the system message they were recorded with.

## What `task_description` becomes

With the flag on, `task_description` describes the task to the teacher in relabelling,
generation, evaluation and judging, and is the system prompt of rows without one. Write it as
a short statement of what every application has in common, not one application's full prompt;
the system messages hold what differs. A one-sentence description works for tool calling
("Control a smart home: turn the user's commands into calls of the available tools."). For a
task with defaults that system messages override, `task_description` states the defaults and
the system messages state the deviations.

A copy of one application's full prompt in `task_description` is copied by the teacher into
the generated examples, and most rows then carry that same prompt. Check for it in the smoke.

## Generation

Without guidance the teacher writes system messages that paraphrase the task description, and
the student learns to ignore them. Add a mutator whose values describe a system message that
changes the correct answer, each ending with "without the system message the answer would be
wrong", plus a second mutator that varies the content (`mutators.md`):

```yaml
synthgen:
  mutators:
    - name: system prompt
      values:
        - "written for an application whose system message states a rule that makes the correct answer a different tool call or different arguments than it would be without the rule; the answer obeys the rule, and without the system message the answer would be wrong"
        - "written for an application whose system message sets a default value the user never mentions; the correct tool call carries that value only because of the system message, and without the system message the answer would be wrong"
        - "written for an application whose system message forbids one tool or one option and names the replacement; the user asks for the forbidden one and the answer uses the replacement, and without the system message the answer would be wrong"
        - "written for an application whose system message defines a nickname or term that the user then uses; the answer resolves it as defined, and without the system message the answer would be wrong"
        - "written for an application whose system message restricts what the assistant may do; the user asks for something outside the restriction and the answer stays within it, and without the system message the answer would be wrong"
    - name: household
      values:
        - "set in a small studio apartment with a single room"
        - "set in a large family house with many rooms and a garage"
        - "set in a shared flat where several people give commands"
```

- Rewrite the values for the task: for redaction, a changed token format, an exempt category, an
  added category, a changed masking rule, a changed `reason` field.
- Keep the closing clause. Milder values ("states a rule the answer must follow") produce
  system messages that most answers ignore.
- Stricter system messages make harder references: read the rows where the answer looks wrong
  before blaming the teacher.

Smoke check, on top of `../stages/synthetic-data-generation.md` Step 4: every new row starts
with a system message, the system messages differ from row to row and from `task_description`,
and in a handful of rows the answer would be different without the system message.

## Evaluation

To check that the answers depend on the system messages, run two teacher evaluations on the
same test-set Dataset: one as configured, and one with `base.variable_system_prompt: false` as
a config override (`execution/cli.md` § Overrides), which replaces every system message by the
fixed prompt. Close scores mean the system messages do not change the answers: fix the test
rows or the mutator values before building the train set. Per row, count the rows the teacher
gets right only with their own system message, and read them to see which kinds of system
message matter.

When matching prediction rows across runs, match on the system and the last user message
together: one user message can appear under several system prompts.

## Deployment

Deployment does not change. The model must run with a system prompt in every request: the
`task_description` it was trained with, or, with `variable_system_prompt` on, the system prompt
the application chooses for that request (`deployment.md` § Why model_client.py instead of raw
requests).
