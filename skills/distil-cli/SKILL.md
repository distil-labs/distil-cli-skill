---
name: distil-cli
metadata:
  version: "13.0.0"
description: >
  Use when building or training a model on the distil labs platform end to end: preparing a
  dataset (config.yaml, job_description.json, traces, train and test rows), running or
  analyzing any pipeline stage (building a test set or a train set from traces by relabelling
  and synthetic data generation, teacher evaluation, model training, deployment), iterating
  when a teacher score or student score is too low, or deciding what to run next after a
  stage completes, or iterating on a model on its own until it reaches a target
  (autoresearch). Activate for phrasings like "build a model for X", "train a student model",
  "build a test set from my traces", "run a teacher eval", "generate synthetic data", "the
  score is low, what now", "add more training data", or "keep iterating until the model is as
  good as you can get it". Stages run through the distil CLI or
  the distil labs API. This skill owns the model-building logic on top of it. Also activate for
  distil labs lookups that are not a full build: distil CLI commands (distil auth,
  distil traces, distil dataset, distil teacher-evaluation, distil slm, distil deployment,
  distil inference-endpoint, distil api-keys, distil access-token), the distil labs REST API,
  config.yaml parameters, supported student and teacher models, what an evaluation metric
  means, or moving a seed dataset or training dataset into a Dataset.
---

# Building Models

Building task-specific small language models on the distil labs platform: from production
traffic or files on disk, through a test set, teacher evaluation and a train set, to a
finetuned, evaluated student served behind an inference endpoint.

Before the first stage, install the `distil` CLI and sign in per
`references/execution/README.md` § Set up the CLI, then settle the backend per § Choose the
backend.

## How the Skill Is Organised

The skill separates what to do from how to run it. Each fact has exactly one owning page, and
every other page links to it.

- **Stages** (`stages/`): one unit of pipeline work each, directly invocable. A stage file is a
  step-by-step procedure: what it needs, what to confirm with the user, how to run it, how to
  analyze the results and, for the stages an iteration re-enters, § What Can Be Changed to
  Improve the Next Iteration. Two kinds: outcome stages (`build-a-test-set.md`,
  `build-a-train-set.md`) say what to aim for and call the mechanics stages
  (`relabel-traces.md`, `synthetic-data-generation.md`), which run one expand operation on one
  split.
- **Workflows** (`workflows/`): sequence the stages and own the decision points, verdicts and
  gates. There are two: `build-a-model.md` builds a model in a single pass, from traffic or
  files to a served model, and `autoresearch.md` iterates on a model for the user, running
  `stages/model-iteration.md` until an agreed target or budget. Each opens with a Workflow
  Map; print it to the user when the workflow starts.
- **References** (`references/`): shared knowledge, one page per topic.
  `references/platform.md` owns how the platform behaves.
- **Execution backends** (`references/execution/`): the only files that know how stages actually
  run. `cli.md` holds every `distil` command and `backend-api.md` every REST API request; stages
  and workflows link their sections, and both files use the same § names, so resolve a citation
  in the backend in use. `references/execution/README.md` owns the setup and the choice: install
  the `distil` CLI and sign in, then use `cli.md` by default or `backend-api.md` when the user
  prefers the API. Make that choice once, before the first stage, and record it in `run.md`.

## The Data Model

One entity holds the data: the **Dataset**, with `config.yaml`, `job_description.json`,
`traces.jsonl`, `train.jsonl` and `test.jsonl`, any data file possibly empty. A Dataset is
uploaded, or created from a traces object, or produced from another Dataset by one of four
expand operations: relabel traces into the test or the train split, or generate synthetic rows
into the test or the train split. Each expand is a new Dataset whose parent is the previous
one, so a project is a chain of Datasets, and a teacher evaluation or a trained model is made
from any Dataset in it (`references/platform.md` § Entities and jobs).

## Stage Protocol

Each stage works in a `<stage-name>/<iteration-id>/` directory under one project root named
after the model, for example `my-model/synthetic-data-generation-train/smoke-1/`. Stages with
a smoke run follow one template:

1. Prepare the Input
2. Confirm the Setup with the User
3. Smoke Run
4. Pull and Analyze the Smoke Outputs
5. Iterate Until the Smoke Passes
6. Full Run
7. Analyze the Results

A smoke run (`smoke-1`, `smoke-2`, then `full-1`) catches config, data and prompt problems
on a subsample before the full run (`references/platform.md` § Smoke runs). The user picks the normal path (smoke first) or the fast path (full run
directly). Stages without a smoke run number their steps consecutively.

Every stage opens with a user gate: present the input, the key config choices, the plan and
the price from a `--dry-run` against the balance (`references/platform.md` § Billing), and get
the user's go-ahead. Nothing launches on defaults the user never saw. After the analysis,
present the findings and agree on the next step together.

A plan the user confirmed covers the stages that carry it out: a stage run from
`stages/model-iteration.md` Step 4 presents only what differs from that plan, and under
`workflows/autoresearch.md` the Step 0 agreement replaces every gate of the stages it runs,
including the agreement after the analysis. Neither covers a decision that
`workflows/autoresearch.md` Step 0 lists as the user's, and every submission is still priced
with `--dry-run`.

The user follows the work in the dashboard, https://app.distillabs.ai, signed in with the
same account as the CLI. When a job starts, give them the page of what it creates:
`/datasets/<id>`, `/slms/<id>`, `/traces/<id>` or `/inference-endpoints/<unique-name>` under
that address. The balance and adding funds are on https://app.distillabs.ai/billing.

Runs are not reproducible: the teacher and the judge run at non-zero temperature, so treat
every score as a sample, and never decide on a difference smaller than the run-to-run variation
measured on the same model and test set (`references/evaluation-metrics.md` § Verdicts).

## File Map

| Stage | Purpose |
|---|---|
| `stages/inference-endpoint.md` | Create an inference endpoint: collect production traces, or serve a trained model to production traffic |
| `stages/build-a-test-set.md` | Fill and lock the test set: relabelled traces, then synthetic rows if needed |
| `stages/build-a-train-set.md` | Fill the train set with the chosen teacher: relabelled traces, then synthetic rows; also a top-up |
| `stages/relabel-traces.md` | Mechanics: one relabelling run into the test or the train split |
| `stages/synthetic-data-generation.md` | Mechanics: one generation run into the test or the train split |
| `stages/teacher-evaluation.md` | Pick the teacher from a shortlist, and score the production model as the baseline |
| `stages/model-training.md` | Finetune the student and evaluate it |
| `stages/model-iteration.md` | Improve a trained model by one change: read the scores, inspect the predictions, find the step the errors come from, change it, run again; keeps the ledger |
| `stages/local-deployment.md` | Serve the trained model with vLLM on your own GPU |

| Workflow | Purpose |
|---|---|
| `workflows/build-a-model.md` | A single pass: endpoint → traces → dataset → test set → teacher evaluation → train set → training → serving endpoint → back to traces. Entry points: an LLM in production, or files on disk |
| `workflows/autoresearch.md` | The agent iterates on a model on its own: `stages/model-iteration.md` again and again, to a target and within a budget agreed once with the user |

| Reference | Purpose |
|---|---|
| `references/platform.md` | How the platform behaves: entities, the expand operations, jobs, smoke runs, overrides, billing, outputs |
| `references/task-types.md` | The four task types and how to choose between them |
| `references/data-preparation/overview.md` | The Dataset directory contract and validation checklist (read first) |
| `references/data-preparation/<task>.md` | Per-task data format: `question-answering.md` (also classifying into labels), `chat-completion.md` (both chat completion tasks) |
| `references/data-preparation/traces.md` | Trace formats, and converting inference endpoint records |
| `references/job-description.md` | Writing job descriptions per task type |
| `references/configuration.md` | config.yaml parameters, defaults, cross-field validation |
| `references/model-catalog.md` | Teacher and student models, defaults, task compatibility |
| `references/reasoning-models.md` | Everything about reasoning students (`base.enable_thinking`) |
| `references/variable-system-prompts.md` | Everything about per-row system prompts (`base.variable_system_prompt`) |
| `references/mutators.md` | Synthetic data diversity controls (`synthgen.mutators`) |
| `references/evaluation-metrics.md` | Metrics per task type, primary metrics, verdicts |
| `references/deployment.md` | Model artifacts, the inference client, local serving |
| `references/inference-endpoints.md` | Inference endpoints: collecting and serving, lifetime, keys, records, the two routes to traces |
| `references/migrating-old-entities.md` | Moving a seed dataset or a training dataset into a Dataset |
| `references/execution/README.md` | Install the CLI, sign in and choose the backend |
| `references/execution/cli.md` | Execution backend: every `distil` command (default) |
| `references/execution/backend-api.md` | Execution backend: every distil labs REST API request |

## Routing

- End-to-end request ("build/train a model for X") → `workflows/build-a-model.md`. Ask what the
  user starts with to pick the entry point: an LLM in production (entry A, Step 1) or files on
  disk, whatever subset of traces, train and test rows (entry B, Step 4).
- Endpoint questions ("put an endpoint in front of GPT-4", "download the traces", "put the
  trained model behind the endpoint") → `stages/inference-endpoint.md`.
- Building evals or a test set ("help me build evals", "I want a test set for my case") →
  `stages/build-a-test-set.md`.
- More or better training data ("generate synthetic data", "add training examples for X",
  "relabel my traces") → `stages/build-a-train-set.md`, or the mechanics stage directly for a
  single run. New rows come from synthetic data generation and relabelling, never from
  external datasets the agent downloads or from rows the agent writes or scripts itself
  (`stages/build-a-train-set.md` § Where New Rows Come From).
- Single-stage request ("run a teacher eval", "train on this dataset") → that stage file plus
  the execution backend in use. If none is set up yet, run
  `references/execution/README.md` § Set up the CLI and § Choose the backend first.
- Results are disappointing ("teacher score is low", "student is far below teacher", "what
  should I change") → `stages/model-iteration.md`, one iteration with the user.
- "Keep iterating until the model is good", "iterate on it for me", a target or a budget for
  improving the model, or autoresearch → `workflows/autoresearch.md`. This wins over the
  end-to-end route when a request asks for both: autoresearch runs `build-a-model.md` first.
- A seed dataset, a training dataset, or a command that exits 1 naming another command →
  `references/migrating-old-entities.md`.
- One model for many system prompts ("per-request system prompt", "multi-tenant prompt",
  "the model should follow the system message", traces with different system prompts) →
  `references/variable-system-prompts.md`.
- Lookup question (a config parameter, a metric, a data format, a command) → the matching
  reference file.
