# Workflow: Build a Model

The outer loop: from the user's traffic or files to a served model, and from the served model's
traffic to the next model. Each step is a stage file: follow the stage's own protocol and
return here for the next step. Before the first step, install the `distil` CLI and sign in per
`../references/execution/README.md` § Set up the CLI, then settle the execution backend per
§ Choose the backend and record it in `run.md`.

## Workflow Map

Print this map to the user when starting the workflow, before the first step:

```
 ENTRY A: an LLM in production                    ENTRY B: files on disk
   ▼                                               (any of traces, train, test)
 1 Collecting endpoint ─── fallback = the production model      │
   ▼                                                            │
 2 Send traffic ─── the application calls the endpoint          │
   │                ... days to weeks ...                        │
   ▼                                                            │
 3 Traces object ─── from the endpoint's records                │
   ▼                                                            │
 4 Dataset ◄────────────────────────────────────────────────────┘
   │   A: from the traces object, with a config and a job description
   │   B: uploaded; the splits the user does not have stay empty
   ▼
 5 Test set ─── relabel traces, then synthetic rows if needed; 100-1000 rows
   │            gate: the user approves it; locked from here on
   ▼
 6 Teacher evaluation ─── the shortlist and the production model, one run each
   │  └─ gate: pick the teacher, PROCEED only
   ▼
 7 Train set ─── relabel some of the remaining traces, then synthetic rows: smoke ► full
   ▼
 8 Train and decide ─┬─ Deploy candidate, above the baseline ─► 9
   │                 └─ Not good enough ─► model-iterations.md (the inner loop)
   ▼
 9 Serving endpoint ─── the student as primary, the production model as fallback
   ▼
10 Send traffic ─── its records are the next model's traces: back to 3, then 4
```

Every Dataset-producing step creates a new Dataset whose parent is the previous one
(`../references/platform.md` § Entities and jobs). Record the chain of ids in `run.md` as it
grows; `distil dataset show` recovers a parent from a child.

## Entry Points

Ask what the user has, then start at the matching step. Both entries meet at Step 4.

| The user has | Entry |
|---|---|
| An LLM in production that OpenRouter serves, and can point the application at an inference endpoint | A, Step 1 |
| Files: a trace file (exported logs, converted), a labelled train set, a test set, or any combination | B, Step 4 |

Prefer entry A whenever the production model is on OpenRouter: Step 9 needs an endpoint anyway,
and an endpoint's fallback must be an OpenRouter model. A user with a trace file and an
OpenRouter production model can do both: upload the file now (B) and put the endpoint in place
for the next iteration.

## Step 1: Provision a Collecting Endpoint

Run `../stages/inference-endpoint.md` Steps 1-5 for a collecting endpoint, with the model the
application calls today as the fallback.

## Step 2: Send Traffic

The user moves the application onto the endpoint (`../stages/inference-endpoint.md` Step 6).
The workflow pauses until the endpoint holds enough traces for Steps 5 and 7: at least
`num_test_relabelled + num_train_relabelled`, 400 at the defaults, plus the generation
context each generation run takes, `T + min(T, 1000)` for a target of T or all that are left
(`../references/platform.md` § The expand operations). At the defaults with no synthetic test
rows, 400 traces give a test set and a train set, and every trace beyond that becomes context
for the train generation. The direct route
sees a call about 24 hours after it was made (`../references/inference-endpoints.md` § From
records to a traces object). Agree with the user when to come back, and stop.

## Step 3: Create a Traces Object

`../stages/inference-endpoint.md` Step 7, direct route: the platform pulls the endpoint's
records into a PreparedTraces. Nothing else is needed yet; the config and the job description
come at Step 4.

## Step 4: Create the Dataset

Pick the task type (`../references/task-types.md`) and write the job description
(`../references/job-description.md`) and the config (`../references/configuration.md`). The
config needs:

- `base.task`;
- `base.teacher_model_name`: the teacher that relabels the traces at Step 5. Default to the
  large GLM 5.3 teacher, `zai.glm-5.3-low-thinking` (`../references/model-catalog.md`
  § Defaults); Step 6 may replace it for Step 7;
- `base.student_model_name`: the student Step 8 will train by default
  (`../references/model-catalog.md` § Defaults), so the chain does not carry the library
  default;
- `evaluation.llm_as_a_judge_model_name`: the judge, set once here and kept for every later run,
  so every score in the project is read by the same judge. Its criteria, including format rules
  such as no code fences, go in `llm_as_a_judge_instructions` in the job description
  (`../references/job-description.md` § What each field feeds);
- `trace_processing.observation_format`: `langfuse` for a traces object from an endpoint,
  `openai_messages` for a converted trace file (`../references/data-preparation/traces.md`).

Then create the Dataset (`../references/execution/cli.md` § The Dataset):

- **Entry A**: `distil dataset create-from-traces` on the traces object. The traces are copied,
  train and test are empty.
- **Entry B**: prepare the directory (`../references/data-preparation/overview.md`), show it to
  the user, and `distil dataset create --data`. The create validates the files; an uploaded
  `test.jsonl` or `train.jsonl` is kept and added to in Steps 5 and 7.

In a later iteration (Step 10), the Dataset is created with the current `test.jsonl` next to
the new traces, so the test set stays the same.

## Step 5: Build the Test Set

Run `../stages/build-a-test-set.md`. The user approves the result: it is the test set every
score in the project is read on, and it is locked from here on. With a complete uploaded test
set, the stage is its review only.

## Step 6: Teacher Evaluation

Run `../stages/teacher-evaluation.md` on the test-set Dataset: one run per candidate teacher,
and one with the production model as teacher, which is the baseline the student must beat.
When the production model has no catalog entry there is no baseline, and Step 8 reads the
student against the teacher and the base student only.

**Gate:** continue only on PROCEED (`../references/evaluation-metrics.md` § Verdicts), with the
best teacher as `base.teacher_model_name` for Step 7.

## Step 7: Build the Train Set

Run `../stages/build-a-train-set.md` on the test-set Dataset with the teacher from Step 6.

## Step 8: Train and Decide

Run `../stages/model-training.md` on the Dataset Step 7 produced, then decide with the user.
Every score is the primary metric agreed for the project
(`../references/evaluation-metrics.md` § Primary metric per task), and the verdicts are in
`../references/evaluation-metrics.md` § Verdicts:

- **Deploy candidate**, and above the production model when there is a baseline → Step 9. For
  a sweep, the smallest student that clears the bar.
- **Anything else** → `model-iterations.md`, which decides whether to retune (Step 8 again on
  the same Dataset), rebuild the train set (Step 7), change the teacher (Step 6), or, with the
  user's approval, change the test set (Step 5). It returns here with a deploy candidate, or
  with a report of why it stopped.

## Step 9: Deploy to an Inference Endpoint

Run `../stages/inference-endpoint.md` Steps 1-5 for a serving endpoint: the student as primary,
the production model as fallback (for entry B without an endpoint, the model the user wants
answering when the student cannot). It is always a new endpoint
(`../references/inference-endpoints.md` § Lifetime).

To run the model on the user's own GPU instead, use `../stages/local-deployment.md`. That ends
the loop: nothing records local traffic.

## Step 10: Send Traffic Through the Model

The user moves the application to the serving endpoint (`../stages/inference-endpoint.md`
Step 6). Its requests must carry the model's system prompt, the job description's
`task_description`, or go through `model_client.py` (`../references/deployment.md` § Why
model_client.py instead of raw requests). Delete the deployment per
`../stages/inference-endpoint.md` Step 8 when the student should stop answering.

The hosted deployment is a session for trying the model (`../references/inference-endpoints.md`
§ Lifetime). Once it ends, the fallback answers every call, so the records after that are the
fallback's answers. For a permanent deployment, the user contacts contact@distillabs.ai; with a
permanent primary, the serving endpoint's records are the student's traffic.

**The transition to the next model.** The records are the next iteration's traces. When the
user wants the model retrained on them:

1. Step 3: a new traces object from the serving endpoint (or the download route, filtered on
   `metadata.source` for the answers the next model should learn from).
2. Step 4: a new Dataset from the new traces, with the current `test.jsonl` and the best
   iteration's `train.jsonl` included (`dataset create --traces --test --train`, after
   `traces download` for the direct route, with `observation_format: langfuse` kept;
   `../references/execution/cli.md` § The Dataset). The test set stays the same and Step 7
   becomes a top-up instead of a rebuild. The old Dataset's leftover traces are not carried.
3. Step 5: keep the test set as it is, or extend it with the new traces. Extending makes every
   earlier score stale (`../stages/build-a-test-set.md` Step 5); it is the right call when the
   failures the user sees in production are not in the test set.
4. Step 6 only when the test set changed; with the same rows, the teacher and the baseline
   scores in `test-set.md` still hold. Then Steps 7 and 8. The previous model's scores are in
   the ledger; with the same test set, the new model is read against them.
