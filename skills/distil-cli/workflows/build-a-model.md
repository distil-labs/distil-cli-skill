# Workflow: Build a Model

The end-to-end loop over the stages. Each step is a stage file: follow the stage's own protocol
and return here for the next step. Before the first step, install the `distil` CLI and sign in
per `../references/execution/README.md` § Set up the CLI.

## Workflow Map

Print this map to the user when starting the workflow, before the first step:

```
 ENTRY A: an LLM in production whose traffic can go through an endpoint
   ▼
 1 Provision a collecting endpoint ─── fallback = the production model, no primary
   ▼
 2 Send traffic ─── the application calls the endpoint; every call is recorded
   │                ... days to weeks, at the rate of the user's traffic ...
   ▼
 3 Create a traces object ─── from the endpoint's records    ◄── ENTRY B: a trace file
   ▼
 4 Test set from traces ─── relabelled test set + original-model baseline
   │                         (first iteration; later iterations skip to 5)
   ▼
 5 Seed dataset from traces ─── trace processing             ◄── ENTRY C: a labelled dataset
   │  └─ gate: teacher evaluation, PROCEED only
   ▼
 6 Synthetic data ─── smoke ► full
   ▼
 7 Train and decide ─┬─ Deploy candidate ─► 8
   │                 ├─ Retune ─► 7 again, no regeneration
   │                 └─ Iterate, or below the original model ─► model-iterations.md
   ▼
 8 Deploy to an inference endpoint ─── serving endpoint: the student as primary,
   │                                    the production model as fallback
   ▼
 9 Send traffic through the model ─── the serving endpoint records it
   └──────────► back to 3, then 5: its records are the next iteration's traces
```

## Entry Points

Ask what the user has, then start at the matching step. Every entry ends in the same loop.

| The user has | Start at |
|---|---|
| An LLM in production that OpenRouter serves, and can point the application at an inference endpoint | Step 1 |
| A file of production traces (the LLM's requests and responses), including exported logs of a production model that OpenRouter does not serve | Step 3 |
| A labelled dataset with a test set, and no production traffic to collect | Step 5 |

Prefer Step 1 whenever the production model is on OpenRouter: Step 8 needs an endpoint anyway,
and an endpoint's fallback must be an OpenRouter model. Entry C needs a test set, because the
teacher-evaluation gate and every verdict are read on it; it has no original-model baseline.

## Step 1: Provision a Collecting Endpoint

Run `../stages/inference-endpoint.md` Steps 1-5 for a collecting endpoint, with the model the
application calls today as the fallback.

## Step 2: Send Traffic

The user moves the application onto the endpoint (`../stages/inference-endpoint.md` Step 6).
The workflow pauses until the endpoint holds enough traces:
`num_traces_to_relabel + 2 × num_synthetic_examples + num_traces_as_training_base`, 400 at the
defaults (`../references/configuration.md` § traces_to_test_set, § trace_processing). The direct
route sees a call about 24 hours after it was made
(`../references/inference-endpoints.md` § From records to a traces object). Agree with the user
when to come back, and stop.

## Step 3: Create a Traces Object

Both routes read `config.yaml` and `job_description.json` from the `--data` directory, so write
them first. Pick the task type (`../references/task-types.md`) and write the job description
(`../references/job-description.md`). The config needs:

- `base.task`;
- `base.teacher_model_name`: the teacher that relabels the traces. Default to the large GLM 5.3
  teacher, `zai.glm-5.3-low-thinking` (`../references/model-catalog.md` § Defaults);
- `evaluation.llm_as_a_judge_model_name`: the judge, set once here and kept for every later run,
  so every score in the project is read by the same judge. Its criteria, including format rules
  such as no code fences, go in `llm_as_a_judge_instructions` in the job description
  (`../references/job-description.md` § What each field feeds);
- `trace_processing.observation_format`: `langfuse` for the direct route, `openai_messages` for a
  converted upload (`../references/data-preparation/traces.md`).

Then create the traces object:

- **From the endpoint (entry A, and every later iteration):** `../stages/inference-endpoint.md`
  Step 7.
- **From a trace file (entry B):** convert it per `../references/data-preparation/traces.md`
  and upload it (`../references/execution/cli.md` § Test set from traces and trace processing).

In a later iteration, put the current `test.jsonl` in the `--data` directory so the test set
stays the same.

## Step 4: Test Set from Traces

First iteration only. Run `../stages/test-set-from-traces.md` on the traces object, with
`traces_to_test_set.evaluate_original_model: true`. The user approves this test set: it gates
every verdict that follows.

Later iterations skip this step and go to Step 5, unless `model-iterations.md` Step 3 says the
test set cannot measure the failure.

## Step 5: Seed Dataset

- **From traces (entries A and B):** run `../stages/trace-processing.md` on the traces object:
  the updated one from Step 4, or the one from Step 3 when Step 4 was skipped.
- **From a labelled dataset (entry C):** prepare the input directory at `seed-dataset/input/`
  under the project root (`../references/data-preparation/overview.md`), show it to the user,
  and create the
  SeedDataset (`../references/execution/cli.md` § The SeedDataset).

**Gate:** run `../stages/teacher-evaluation.md` on the SeedDataset and continue only on PROCEED
(`../references/evaluation-metrics.md` § Verdicts).

## Step 6: Synthetic Data

Run `../stages/synthetic-data-generation.md` on the SeedDataset with the teacher that passed
evaluation.

## Step 7: Train and Decide

Run `../stages/model-training.md` on the TrainingDataset, then decide with the user. Every score
is the primary metric agreed for the project (`../references/evaluation-metrics.md` § Primary
metric per task),
and the verdicts are in `../references/evaluation-metrics.md` § Verdicts:

- **Deploy candidate** → Step 8. For a sweep, the smallest student that clears the bar.
- **Retune** → Step 7 again on the same TrainingDataset, with another student or tuning.
- **Iterate**, or below the original model → `model-iterations.md`.

## Step 8: Deploy to an Inference Endpoint

Run `../stages/inference-endpoint.md` Steps 1-5 for a serving endpoint: the student as primary,
the production model as fallback (for entry C, the model the user wants answering when the
student cannot). It is always a new endpoint (`../references/inference-endpoints.md` § Lifetime).

To run the model on the user's own GPU instead, use `../stages/local-deployment.md`. That ends
the loop: nothing records local traffic.

## Step 9: Send Traffic Through the Model

The user moves the application to the serving endpoint (`../stages/inference-endpoint.md`
Step 6). Its requests must carry the model's system prompt, the job description's
`task_description`, or go through `model_client.py` (`../references/deployment.md` § Why
model_client.py instead of raw requests). Delete the deployment per `../stages/inference-endpoint.md` Step 8 when
the student should stop answering.

The hosted deployment is a session for trying the model (`../references/inference-endpoints.md`
§ Lifetime). Once it ends, the fallback answers every call, so the records after that are the
fallback's answers. For a permanent deployment, the user contacts contact@distillabs.ai; with a
permanent primary, the serving endpoint's records are the student's traffic. The records are the
next iteration's traces: back to Step 3, then Step 5, with `model-iterations.md` deciding what to
change.
