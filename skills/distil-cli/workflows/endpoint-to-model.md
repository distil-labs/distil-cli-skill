# Workflow: Endpoint to Model

The default end-to-end workflow. It starts before any data exists: an inference endpoint goes in
front of the model the user already runs in production, records the traffic, and the recorded
traces become the seed data. It ends with the trained student serving that same traffic through a
new endpoint, which records again, so the next iteration starts where this one ended.

Take this workflow whenever the user runs an LLM in production, or is about to, and does not
already hold a trace file or a labeled dataset. A user with a trace file starts at
`traces-to-model.md`; a user with a labeled dataset and no production traffic at
`dataset-to-model.md`.

Before Step 1, settle the execution backend: install the `distil` CLI per
`../references/execution/README.md` § Choose the backend, and record the result in `run.md`.

## Workflow Map

Print this map to the user when starting the workflow, before the first step:

```
1 Collect ─── inference endpoint in front of the production model; the app calls it
      │       (the fallback answers as before; every call is recorded)
      │       ... traffic accumulates: hours to weeks, not minutes ...
      ▼
2 Download and Convert ─── endpoint records ► traces.jsonl in observation format
      ▼
3 Trace Processing ─── traces.jsonl ► seed train/test + original-model baseline
      ▼
4 Teacher Eval ──► 5 Synthetic Data ──► 6 Training ──► 7 Decide
└──────────────── dataset-to-model.md Steps 2-5 ────────────────┘
                  extra floor at Decide: below the original model, never serve
      ▼
8 Serve Behind the Endpoint ─── deploy the student; new endpoint with it as primary
      │                         and the production model as fallback; the app moves over
      └──────────────► back to 1: the new endpoint records the student's traffic
```

## Step 1: Collect

Create the endpoint, link a key, and have the user point their application at it: the execution
backend in use, § Inference endpoints. Three things to settle with the user before creating it:

- **The fallback model** is the model their application calls today, as an OpenRouter slug. Ask.
  The endpoint must answer exactly as production does, or the recorded traces describe a
  different system.
- **The trace sample rate.** Default 1, every call is recorded. Lower it only for very high
  traffic, and say what it does to the collection time.
- **The system prompt stays in the application.** The endpoint records the full request, and
  trace processing strips the system prompt from the traces and expects its content in the job
  description instead (Step 3). Nothing changes in the application except the base URL, the key
  and the model name.

Send one request through the endpoint together, as in the backend's § The call the user has to
make, before the application moves. Then this workflow pauses. Trace processing wants hundreds
of traces, and they arrive at the rate of the user's traffic. Agree when to come back, record the
unique endpoint name and the key's file in `run.md`, and stop.

## Step 2: Download and Convert

Download the records (the backend § Download the traces) and convert them into the observation
format: `../references/data-preparation/traces.md` § From an inference endpoint has the record
shape and the conversion. Check the converted line count against the download and read a few
lines by eye. This step is data preparation, not a platform job, so it has no stage file.

## Step 3: Trace Processing

Run `../stages/trace-processing.md` on the converted file. The job description's
`task_description` must mirror the system prompt the application sends, since the traces lose it.
Review the generated test set and the original-model baseline with the user before accepting the
full run. Then continue with the trace-specific rules of `traces-to-model.md` § Steps 2-6.

## Steps 4-7: Continue as Dataset to Model

Follow `dataset-to-model.md` from Step 2 (teacher evaluation) through Step 5 (decide). The
original-model baseline is a floor at the decision: a student that does not beat the model it
replaces is not served, whatever `closed` says.

## Step 8: Serve Behind the Endpoint

Run `../stages/model-deployment.md` and take its route "Behind the inference endpoint": deploy
the student, smoke-test the deployment directly, then create a new endpoint with the deployment
as primary and the production model as fallback, and move the application's `model` string to
the new endpoint name. The endpoint cannot be edited after creation, so this is always a new
endpoint, and the old one keeps recording until the application moves.

Two things to say to the user at this step:

- The hosted deployment is for trying the student on live traffic. It stops on its own, and once
  it does every call goes to the fallback. A permanent endpoint is a request to
  contact@distillabs.ai.
- The new endpoint records too, and its records name which model answered each call. Those are
  the traces for the next iteration, so the workflow loops back to Step 1 with the student in
  place of the production model.
