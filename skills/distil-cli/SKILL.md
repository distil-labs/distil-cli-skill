---
name: distil-cli
metadata:
  version: "10.0.3"
description: >
  Use when building or training a model on the distil labs platform end to end: preparing a
  dataset (config.yaml, job_description.json, traces, train and test rows), running or
  analyzing any pipeline stage (building a test set or a train set from traces by relabelling
  and synthetic data generation, teacher evaluation, model training, deployment), iterating
  when a teacher score or student score is too low, or deciding what to run next after a
  stage completes. Activate for phrasings like "build a model for X", "train a student model",
  "build a test set from my traces", "run a teacher eval", "generate synthetic data", "the
  score is low, what now", or "add more training data". Stages run through the distil CLI or
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
  gates. `build-a-model.md` is the outer loop, from traffic to a served model and back;
  `model-iterations.md` is the inner loop that improves a model on a locked test set. Each
  opens with a Workflow Map; print it to the user when the workflow starts.
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
the expected cost against the remaining credits (`references/platform.md` § Credits), and get
the user's go-ahead. Nothing launches on defaults the user never saw. After the analysis,
present the findings and agree on the next step together.

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
| `stages/local-deployment.md` | Serve the trained model with vLLM on your own GPU |

| Workflow | Purpose |
|---|---|
| `workflows/build-a-model.md` | The outer loop: endpoint → traces → dataset → test set → teacher evaluation → train set → training → serving endpoint → back to traces. Entry points: an LLM in production, or files on disk |
| `workflows/model-iterations.md` | The inner loop: read the scores, inspect the predictions, find the step the errors come from, change it, run again. Runs autonomously to an agreed target on a locked test set |

| Reference | Purpose |
|---|---|
| `references/platform.md` | How the platform behaves: entities, the expand operations, jobs, smoke runs, overrides, credits, outputs |
| `references/task-types.md` | The four task types and how to choose between them |
| `references/data-preparation/overview.md` | The Dataset directory contract and validation checklist (read first) |
| `references/data-preparation/<task>.md` | Per-task data format: `question-answering.md`, `classification.md`, `chat-completion.md` (both chat completion tasks) |
| `references/data-preparation/traces.md` | Trace formats, and converting inference endpoint records |
| `references/job-description.md` | Writing job descriptions per task type |
| `references/configuration.md` | config.yaml parameters, defaults, cross-field validation |
| `references/model-catalog.md` | Teacher and student models, defaults, task compatibility |
| `references/reasoning-models.md` | Everything about reasoning students (`base.enable_thinking`) |
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
  single run.
- Single-stage request ("run a teacher eval", "train on this dataset") → that stage file plus
  the execution backend in use. If none is set up yet, run
  `references/execution/README.md` § Set up the CLI and § Choose the backend first.
- Results are disappointing ("teacher score is low", "student is far below teacher"), or "keep
  iterating until the model is good" → `workflows/model-iterations.md`.
- A seed dataset, a training dataset, or a command that exits 1 naming another command →
  `references/migrating-old-entities.md`.
- Lookup question (a config parameter, a metric, a data format, a command) → the matching
  reference file.
