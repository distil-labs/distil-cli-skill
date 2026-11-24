---
name: distil-cli
metadata:
  version: "16.0.0"
description: >
  Use when building, training, improving or evaluating a task-specific small language model on
  the distil labs platform, end to end or one step at a time: scoping the model with the user,
  writing or tuning the task description, building or expanding a test set, generating or
  relabelling training data, training a student, iterating until the model reaches the teacher,
  and deploying and evaluating it. Activate for phrasings like "build a model for X", "replace
  our GPT calls with a small model", "train a student model", "build a test set from my
  traces", "run a teacher eval", "generate synthetic data", "add more training data", "the
  score is low, what now", "train an OCR model on my scans", or "keep iterating until the model
  is as good as it gets". Also
  activate for distil labs lookups: distil CLI commands (distil auth, traces, dataset,
  teacher-evaluation, slm, deployment, inference-endpoint, api-keys, access-token), the distil
  labs REST API, config.yaml parameters, supported student and teacher models, evaluation
  metrics, or moving a seed dataset or training dataset into a Dataset.
---

# Building Models

This skill builds a task-specific small language model (the **student**) on the distil labs
platform. A large model (the **teacher**) writes the training data, and the student learns from
it. The work follows one workflow, below: agree the scope, fix the task description, build a
test set you can trust, then iterate on the training data until the student reaches the
teacher, and finally deploy and evaluate it.

Before the first platform call, install the `distil` CLI and sign in
(`references/execution/README.md` § Set up the CLI), and choose the backend: the CLI
(`references/execution/cli.md`) by default, or the REST API (`references/execution/backend-api.md`)
when the user prefers it (§ Choose the backend). Record the choice in the run log.

## How the skill is organised

The skill is layered like a program. It ships no scripts: the protocols say what to compute, and
you write the code each task needs, because every task's data format and metric differ. This file is the main logic: the order of the phases,
every user gate, the iteration loop and the rules. Each other file is read only when the
workflow needs it, and links only to the layers below it.

| Layer | Contents | Read it when |
|---|---|---|
| `phases/` | One file per phase: its steps and decisions | the workflow enters that phase |
| `protocols/` | Routines several phases share: making rows, blind review, error analysis, the run log | a phase calls one |
| `operations/` | One platform job each: configure, price, submit, wait, pull. No decisions | before running that job |
| `references/` | Facts, one owner each: config, mutators, metrics, models, formats, billing | you need a specific fact |
| `references/execution/` | Every CLI command and REST request | you run a command |

Phases never link to each other. They share data through the **run log**
(`protocols/run-log.md`): each phase reads what earlier phases wrote there and writes its own
outputs back. This keeps every phase readable on its own, and lets the work resume after a
context reset.

## The data model

One entity holds the data: the **Dataset**, with `config.yaml`, `job_description.json`,
`traces.jsonl`, `train.jsonl` and `test.jsonl` (any data file may be empty). A Dataset is
uploaded, created from a traces object, or produced from another Dataset by one of four expand
operations: relabel traces into the test or the train split, or generate synthetic rows into the
test or the train split. Each expand creates a new Dataset whose parent is the previous one, so
a project is a chain of Datasets. A teacher evaluation or a trained model is made from any
Dataset in the chain (`references/platform.md` § Entities and jobs).

## Rules

These hold in every phase. Each comes with its reason, so you can apply it to cases it does not
name.

- **Make and fix data on the platform first; fix it locally when that helps.** Rows come from
  the user's files, from relabelling or from synthetic generation. When rows are wrong, the first
  fix is the setting that produced them: the config, the mutators, the generation instructions,
  the LLM validator, or the job description, because that fixes every later run too. When a
  setting cannot fix it, or a local step is simpler (for example dropping generated rows whose
  JSON does not parse, or correcting a label the review proved wrong), download the Dataset, fix
  or filter the rows with a script, and upload the result as a new Dataset. This is a normal
  option, not an exception: it needs no approval. Record the script, what it changed and the row
  counts in the run log, because the upload has no parent on the platform and the log is then the
  only record of how its rows were made.
- **Never add rows from outside the platform.** No downloaded datasets, no scripted or
  hand-written rows, unless the user asks. A rule you would encode in a script goes into the
  generation instructions or a mutator, so the teacher writes the rows.
- **One change per iteration.** Otherwise a score change cannot be traced to its cause. Smoke
  runs that repair the change itself are part of the same change.
- **Training data only grows.** Each iteration adds rows to the best Dataset so far, because
  more correct data helps the student generalise. When an iteration makes the model noticeably
  worse, fork the next iteration from the earlier Dataset instead.
- **Price every submission with `--dry-run` first** and add the price to the spending in the
  run log before submitting (`references/platform.md` § Billing). The budget is the user's, and
  the dry run is the only way to see a price before it is charged.
- **Every score is a sample.** The teacher and the judge run at non-zero temperature, and two
  trainings on the same data give different models. Never act on a difference smaller than the
  noise that applies to it, and say which noise you used (`references/evaluation-metrics.md`
  § Verdicts).
- **Cancel a job only on the user's say-so, except your own mistakes in Phase 4.** A submitted
  teacher evaluation, dataset expansion or training can be cancelled until it finishes
  (`references/execution/cli.md` § Cancel a job); a cancel soon after submitting is refunded
  (`references/platform.md` § Billing). Offer it when a submission turns out wrong or a job stays
  pending. Inside Phase 4, where the user is not involved, cancel your own submission as soon as
  you find it wrong (wrong parent, config or rows), and report it at the next stop.
- **Log every id, score and decision** in the run log as it happens (`protocols/run-log.md`).
  The log is how you, the user, or a later session can tell what was done and why.
- **Show the user where to look.** When a job starts, give them its dashboard page under
  https://app.distillabs.ai: `/datasets/<id>`, `/slms/<id>`, `/traces/<id>` or
  `/inference-endpoints/<unique-name>`. Balance and funds are on https://app.distillabs.ai/billing.

## The workflow

```
 Entry A: inference endpoint            Entry B: dataset upload
        └──────────────┬──────────────────────┘
 0 Start ─── the user's traffic or files → first Dataset
 1 Define the task ─── inputs, outputs, scope → draft job_description.json  ◆ user confirms
 2 Plan the build ─── teacher, student(s), budget, metric; sanity check   ◆ user agrees
 3 Test set ─── real rows, checked against the scope; fill gaps → target, noise ◆ user agrees
 4 Model iterations ─── add rows → train → compare → analyse → repeat
        stop: target reached · plateau · budget spent · user says stop
 5 Deploy and evaluate ─── production settings, against the other setups ◆ user decides to serve
```

Print this map to the user when the workflow starts. ◆ marks a gate: the work does not continue
until the user approves. Between the gates, work without asking.

**Inputs with images.** When the model reads images (scans, photos, screenshots), the workflow
changes: there is no synthetic data generation, and every model must be a vision model. Read
`references/visual-tasks.md` before the first upload in Phase 0, and follow its changes to each
phase.

### Phase 0: Start

Read `phases/0-start.md`. Ask what the user has. **Entry A**: an LLM in production that can sit
behind a distil labs inference endpoint, whose traffic becomes traces. **Entry B**: files to
upload (labelled rows, train rows, unlabelled inputs). Create the project directory and the run
log, and the first Dataset.

Output: the first Dataset id; whether traces, labelled test rows and train rows exist.

### Phase 1: Define the task

Read `phases/1-define-the-task.md`. Goal: understand the task well enough to write it down. Users
give this in different ways (a written spec, files, a conversation): read what they gave, then ask
for what is missing. Every rule comes from the user; record unknowns as open questions. It
produces:

- **the scope:** the job and which mistakes are costly, an **input inventory** of everything
  production sends (common and rare), the out-of-scope inputs, the outputs and what is correct,
  the task type, and the available data;
- **a draft job description that defines the task:** `task_description`, the judge rubric
  (`llm_as_a_judge_instructions`), and `tools` when the model calls tools. The data-making fields
  (generation and relabelling instructions) are written later, in Phases 3 and 4.

Output: the scope in the run log, the draft job description. ◆ The user confirms them.

### Phase 2: Plan the build

Read `phases/2-plan-the-build.md`. Goal: agree how the model is built, then check the job
description and agree it with the user. The agreement covers:

- teacher and student(s);
- budget and time, priced with dry runs;
- the metric: deterministic when the task has one, otherwise the judge;
- the baseline, thresholds and run sizes.

When the user provided a test set, one teacher evaluation on it is a sanity check of the job
description, shown to the user. It is not a tuning step, because Phase 3 builds the real test set
next.

Output: the agreement, the agreed job description, and the root Dataset that carries it. ◆ The
user agrees to both. Once agreed, the job description is not changed.

### Phase 3: Test set

Read `phases/3-test-set.md`. Goal: a test set that covers the scope. Start from the user's
labelled rows and relabelled traces, then check them against the scope item by item: input
kinds, outputs, rules, thresholds and out-of-scope inputs. ◆ Propose the gaps to the user, then
fill them with rows built for each gap (relabelled traces, or generation with settings written
for that gap, never the training settings), through `protocols/make-rows.md`. Then two teacher
evaluations on the agreed test set give the evaluation noise, and evaluations of one to three
other strong models set the target (their mean).

Output: the test-set Dataset, its coverage table, the target and the noise. ◆ The user
agrees to the test set. Once agreed, it is not changed.

### Phase 4: Model iterations

Read `phases/4-model-iteration.md`. One iteration:

1. scope the rows to add, from the last error analysis (iteration 1: the initial train set,
   from relabelled traces when there are traces, otherwise generated);
2. make them with `protocols/make-rows.md` on the best Dataset so far: smoke, blind review, fix
   until it passes, then the full run;
3. train;
4. compare with the best model, on the group the change aimed at and on the real-row groups:
   keep the new Dataset when the targeted group clearly improved and nothing that matters got
   worse beyond the noise; otherwise roll back. Close calls are a judgement: measure the training
   noise before you decide (`phases/4-model-iteration.md` Step 4);
5. error analysis (`protocols/error-analysis.md`): the areas the model must improve on, which
   scope the next iteration.

The user is not involved in the loop. After each iteration, stop when:

- **the target is reached**: the best score is above the target, or below it by no more than the
  noise;
- **it plateaus**: two iterations in a row within the noise of the best score;
- **the budget is spent**: the next iteration (rows plus training) would go over the budget, or
  the iteration cap is reached;
- **the user says stop;**
- **the next step needs the user**: the largest remaining finding is a scope gap or another
  change to the test set (`protocols/error-analysis.md` § Output: findings). Smaller scope gaps wait for
  the next stop while the loop continues on the other findings.

Read the target and the plateau on the row groups that still separate models. A group where the
student scores at the ceiling shows no further progress, so an overall score that passes the
target because of such a group does not count: the real-row groups and the groups below the
ceiling decide.

At every stop, set Phase 4 to done in the run log header, and report from the ledger: the
target, the best model and its scores, what each iteration changed and what it did, the money
spent, and why the loop stopped.

### Phase 5: Deploy and evaluate

Read `phases/5-deploy-and-evaluate.md`. Deploy the best model and evaluate it with the
production inference settings, against the other setups: the base student, the teacher and the
production model. Report where the gains came from and what is left. ◆ The user decides whether
to serve it, behind a serving endpoint or on their own GPU. The serving endpoint's traffic can
start the next cycle.

### Resuming

Before anything else, look for a run log (`protocols/run-log.md`). If one exists, read it: the
first phase whose output is missing is where the work continues. A request such as "keep
improving the model" enters at Phase 4 when Phases 0–3 are done. If a needed output is missing
(for example no agreed budget or no agreed test set), run that phase first.

## Other requests

- **A single platform job** ("run a teacher eval", "train on this Dataset", "deploy this model")
  → the matching file in `operations/`. Still price it with a dry run and log it.
- **A test set only** ("help me build evals") → Phases 0–3.
- **A lookup** (a config parameter, a command, a metric, a model) → the reference in the file
  map below.
- **One model for many system prompts** → `references/variable-system-prompts.md`.
- **A seed dataset, a training dataset, or a command that exits 1 naming another command** →
  `references/migrating-old-entities.md`.

## File map

| File | Read it when |
|---|---|
| `phases/0-start.md` | starting a project: entry points, first Dataset |
| `phases/1-define-the-task.md` | understanding the task: scope, input inventory, and the draft job description |
| `phases/2-plan-the-build.md` | agreeing teacher, students, budget, metric, thresholds; checking and agreeing the job description |
| `phases/3-test-set.md` | checking the test set's coverage against the scope, filling gaps, agreeing it; target and noise |
| `phases/4-model-iteration.md` | running one model iteration |
| `phases/5-deploy-and-evaluate.md` | evaluating the best model with production settings; serving it |
| `protocols/make-rows.md` | adding rows to a split: smoke, review, fix, full run |
| `protocols/blind-review.md` | verifying new rows with independent subagents |
| `protocols/error-analysis.md` | reading a model's (or the teacher's) errors to find what to change |
| `protocols/run-log.md` | the project directory, the run log, lineage and ledger; resuming |
| `operations/create-dataset.md` | uploading files, or creating a Dataset from traces |
| `operations/relabel-traces.md` | one relabelling run into the test or train split |
| `operations/generate-synthetic-data.md` | one synthetic generation run into the test or train split |
| `operations/teacher-evaluation.md` | scoring a model as teacher on the test set |
| `operations/train-model.md` | training a student and reading its scores |
| `operations/inference-endpoint.md` | collecting or serving endpoints, keys, traces |
| `operations/evaluate-deployment.md` | deploying a trained model and running a split through it |
| `operations/local-deployment.md` | serving a trained model with vLLM on the user's GPU |
| `references/platform.md` | entities, expand operations, job status, smoke runs, overrides, billing, outputs |
| `references/task-types.md` | the task types and how to choose one |
| `references/data-preparation/overview.md` | the Dataset directory contract (read first for uploads) |
| `references/data-preparation/<task>.md` | per-task data format: `question-answering.md`, `chat-completion.md` |
| `references/data-preparation/traces.md` | trace formats; converting endpoint records |
| `references/job-description.md` | writing job descriptions per task type; judge instructions |
| `references/configuration.md` | config.yaml parameters, defaults, cross-field validation |
| `references/model-catalog.md` | teacher and student models, defaults, task compatibility |
| `references/reasoning-models.md` | reasoning students (`base.enable_thinking`) |
| `references/variable-system-prompts.md` | per-row system prompts (`base.variable_system_prompt`) |
| `references/visual-tasks.md` | image inputs (`base.visual_task`) and every change they make to the workflow |
| `references/mutators.md` | diversity controls for generation (`synthgen.mutators`) and patterns that work |
| `references/evaluation-metrics.md` | metrics per task, the default metric, noise, verdicts |
| `references/deployment.md` | model artifacts, the inference client, local serving |
| `references/inference-endpoints.md` | endpoints: collecting and serving, lifetime, keys, records |
| `references/migrating-old-entities.md` | moving old seed or training datasets into a Dataset |
| `references/execution/README.md` | installing the CLI, signing in, choosing the backend |
| `references/execution/cli.md` | every `distil` command |
| `references/execution/backend-api.md` | every REST API request |
