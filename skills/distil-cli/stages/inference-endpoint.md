# Stage: Inference Endpoint

Creates an inference endpoint and moves the user's application onto it. One stage covers both
kinds of endpoint (`../references/inference-endpoints.md` § What an endpoint is):

- **Collecting endpoint**: sits in front of the model the user runs in production; its records
  become the traces the first model is built from.
- **Serving endpoint**: sits in front of a hosted deployment of a trained SLM, with the
  production model as the fallback; its records are the traces of the next iteration.

Both are created and called the same way; the serving endpoint adds a deployment before it and
a deployment to delete at the end. To serve a model on the user's own GPU instead, use
`local-deployment.md`.

## Working Directory

```
inference-endpoint/
└── <endpoint-prefix>/
    ├── run.md           # kind, fallback, unique endpoint name, key name, and for a serving
    │                    # endpoint the SLM id and the deployment id
    └── analysis.md      # smoke-test results
```

## Step 1: Prepare the Input

Choose the kind. A collecting endpoint needs the production model's OpenRouter slug; a serving
endpoint needs that slug and a trained SLM id. An endpoint cannot be edited, so a serving
endpoint is always a new endpoint, never the collecting one
(`../references/inference-endpoints.md` § Lifetime).

## Step 2: Confirm the Setup with the User

Present and confirm:

- **The fallback model**: the model the application calls in production today. Ask rather than
  guess.
- **The name prefix.** The platform appends a suffix to it.
- **The trace sample rate**: default 1, every call; lower it only for very high traffic. It is
  fixed at creation.
- **The key**: an existing inference API key or a new one. The secret goes into the user's
  secret store, never into the transcript or `run.md`.
- **The credits** for the kind chosen (`../references/platform.md` § Credits).
- **For a serving endpoint**: which SLM, and that its deployment is a session for trying the
  model: it stops on its own, after which the fallback answers every call. A permanent
  deployment is a request to contact@distillabs.ai (`../references/inference-endpoints.md`
  § Lifetime).

## Step 3: Deploy the SLM (serving endpoint only)

Create the deployment and poll `deployment_status` to `JOB_SUCCESS`
(`../references/execution/cli.md` § Deploy an SLM). `JOB_SUCCESS` means vLLM is answering.
Record the deployment id in `run.md`.

## Step 4: Create the Endpoint and Link the Key

Create the endpoint, with no primary for a collecting endpoint or from the deployment for a
serving endpoint (`../references/execution/cli.md` § Create an endpoint). Create a key if needed
(`../references/execution/cli.md` § Keys) and link it to the endpoint with `link-api-key`
(`../references/execution/cli.md` § Inference endpoints). Record the unique endpoint name in
`run.md`.

## Step 5: Smoke Test

Send one request through the endpoint before the application moves, so a failure is a request
the user can read rather than a production incident (`../references/execution/cli.md` § The call
the application makes). A newly linked key can be rejected for a short time
(`../references/inference-endpoints.md` § Keys).

For a serving endpoint, also send a handful of test-set rows through the model's own
`model_client.py`, pointed at the endpoint, and compare the answers with the expected ones
(`../references/deployment.md` § Why model_client.py instead of raw requests). Then download the
endpoint's records and confirm `metadata.source` reads `primary` for those calls. A `fallback`
answer came from the production model, which usually means the deployment failed or has
stopped. Record the results in `analysis.md`.

## Step 6: Move the Application

The application changes three strings in its OpenAI client: the base URL to
`https://inference.distillabs.ai/v1`, the API key to the linked key, and `model` to the unique
endpoint name. Everything else stays the same, the system prompt included
(`../references/data-preparation/traces.md` § Conversion guidance).

When a serving endpoint replaces a collecting one, only the `model` string changes, because the
same key can be linked to both. Requests to a serving endpoint must carry the model's system
prompt, the job description's `task_description`: the application sends it as its system prompt,
or calls through `model_client.py` (`../references/deployment.md` § Why model_client.py instead
of raw requests). The old endpoint keeps recording until the application stops calling it.

## Step 7: Turn the Records into a Traces Object

Traces accumulate at the rate of the user's traffic, and the next stages want hundreds, so this
step comes days or weeks after Step 6. Choose between the two routes
(`../references/inference-endpoints.md` § From records to a traces object):

- **Directly from the endpoint** (`../references/execution/cli.md` § Traces from an endpoint),
  for the full trace set.
- **Download, convert, upload** (`../references/data-preparation/traces.md` § From an inference
  endpoint), when the traces are needed now or must be filtered first, for example on
  `metadata.source`.

The result is a PreparedTraces, the input of `test-set-from-traces.md`.

## Step 8: Delete the Deployment (serving endpoint only)

Delete the deployment when the user no longer wants the student answering
(`../references/execution/cli.md` § Deploy an SLM), and note it in `run.md`. The endpoint stays
and answers from the fallback.
