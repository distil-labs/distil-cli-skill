# Operation: Inference Endpoint

Creates an inference endpoint, links an API key to it, and turns its records into traces. One
operation covers both kinds (`../references/inference-endpoints.md` § What an endpoint is):

- **Collecting endpoint**: no primary; the production model, as the fallback, answers every call.
  Its records become the traces the first model is built from.
- **Serving endpoint**: a hosted deployment of a trained SLM as the primary, the production model
  as the fallback. Its records are the traces of the next cycle.

| | |
|---|---|
| **Called by** | the start phase (collecting endpoint) and the deploy phase (serving endpoint) |
| **Price** | the endpoint, keys and traces are free. A serving endpoint needs a deployment: priced by `distil deployment create-from-slm --dry-run`, charged at creation, not refunded when it fails, and charged again for every replacement (`../references/platform.md` § Billing) |

## Settings that control the output

| Setting | Where | What it changes |
|---|---|---|
| `--fallback-model` | create | The model that answers when there is no primary or the primary fails: the model the application calls in production today, as an OpenRouter slug in `owner/model` form. When OpenRouter does not serve that model, the closest model it does serve |
| `--name` | create | A prefix; the platform appends a suffix and returns the unique endpoint name every command and request uses |
| `--trace-sample-rate` | create | The fraction of calls recorded, default 1 (every call) in the CLI. Lower it only for very high traffic. Fixed at creation |
| the SLM id | deployment | Which trained model is the primary (serving only). An SLM trained with a `lora_r` outside 8, 16, 32, 64, 128, 256, 320, 512 cannot be deployed |

An endpoint is permanent: it cannot be edited or deleted, and has no idle timeout. Changing the
fallback or the primary means a new endpoint with a new unique name, so a serving endpoint is
always new, never the collecting one. The deployment behind a primary stops after six hours, or
after one hour with no traffic, and cannot be restarted; from then on the fallback answers every
call. A permanent deployment is a request to contact@distillabs.ai
(`../references/inference-endpoints.md` § Lifetime).

## Working directory

```
inference-endpoint/
└── <prefix>/
    └── run.md     # kind, fallback, unique endpoint name, key name (never the secret),
                   # and for a serving endpoint the SLM id, deployment id and smoke-test results
```

## Step 1: Prepare the input

Collect the fallback's OpenRouter slug (ask the user which model the application calls today
rather than guessing; when OpenRouter does not serve it, agree the closest model it does serve), the name prefix, and the key to use. An account holds at most 5 inference
API keys and one key can be linked to many endpoints, so reuse an existing key when there is one
(`distil api-keys list --output json`). A serving endpoint also needs the trained SLM id.

## Step 2: Price it

A collecting endpoint is free. For a serving endpoint, price the deployment and add it to the run
log's spending:

```bash
distil deployment create-from-slm --dry-run <slm-id>
```

## Step 3: Submit

**Serving only: deploy the SLM** (`../references/execution/cli.md` § Deploy an SLM):

```bash
distil deployment create-from-slm --output json <slm-id> | jq -r .id
bash poll.sh deployment <deployment-id>
```

`deployment status` answers `{"deployment_status": …, "endpoint_status": …}`; it reads
`JOB_RUNNING` while starting, and `JOB_SUCCESS` means vLLM is answering. Record the deployment id
in `run.md` and the run log's lineage.

**Create the endpoint** (`../references/execution/cli.md` § Create an endpoint):

```bash
distil inference-endpoint create --output json --name <prefix> --fallback-model "<owner/model>" | jq -r .unique_endpoint_name
distil inference-endpoint create-from-deployment --output json --name <prefix> \
  --fallback-model "<owner/model>" <deployment-id> | jq -r .unique_endpoint_name
```

`create-from-deployment` refuses with `Deployment is not ready` before `JOB_SUCCESS`. Record the
unique endpoint name, and give the user
https://app.distillabs.ai/inference-endpoints/<unique-endpoint-name>.

**Create a key if needed, and link it** (`../references/execution/cli.md` § Keys):

```bash
distil api-keys create <key-name> > /dev/null      # writes <key-name>.json in the current directory
chmod 600 <key-name>.json
distil inference-endpoint link-api-key <unique-endpoint-name> <key-name>
```

The secret is printed once and kept in the file; the output goes to `/dev/null` so it stays out of
the transcript. Never echo it, and never write it into `run.md` or the run log: name the file. It
goes into the user's secret store for the application.

**Smoke test.** Send one request through the endpoint before the application moves, so a failure
is a request you can read rather than a production incident
(`../references/execution/cli.md` § The call the application makes). A newly linked key can be
rejected for up to a minute: wait and retry. For a serving endpoint, also send a handful of
test-set rows through the model's own `model_client.py` pointed at the endpoint, compare the
answers with the references, download the records, and confirm `metadata.source` reads
`primary` for those calls (`../references/deployment.md` § Serving hosted). A `fallback` answer
came from the production model, which usually means the deployment failed or has stopped. Record
the results in `run.md`.

**Move the application.** The application changes three strings in its OpenAI client: the base
URL to `https://inference.distillabs.ai/v1`, the API key to the linked key, and `model` to the
unique endpoint name. Everything else stays the same. When a serving endpoint replaces a
collecting one, only `model` changes, because the same key can be linked to both. Requests to a
serving endpoint must carry the system prompt the model was trained with (the `task_description`,
or with `base.variable_system_prompt` on, the prompt the application chooses), with the sampling
and thinking settings of `model_client.py` (`../references/deployment.md` § Why model_client.py
instead of raw requests). The old endpoint keeps recording until the application stops calling it.

## Step 4: Wait and pull

Traces accumulate at the rate of the user's traffic, so this comes days or weeks after the move.
Two routes turn the records into traces (`../references/inference-endpoints.md` § From records to
a traces object):

- **Directly**: `distil traces create-from-inference-endpoint <unique-endpoint-name>` makes a
  traces object from every record (`../references/execution/cli.md` § Traces from an endpoint). A
  call becomes available about 24 hours after it was made. The Dataset made from it needs
  `observation_format: langfuse`.
- **Download, convert, upload**: download the records and convert them
  (`../references/data-preparation/traces.md` § From an inference endpoint), for traces needed now,
  a few thousand at most, or records that must be filtered first:

```bash
distil inference-endpoint download-traces --from-start-time <date> --all \
  --file-name inference-endpoint/<prefix>/records.jsonl <unique-endpoint-name>
```

Without `--from-start-time` the window is the last day; without `--all`, at most `--count`
records (default 1000). With no records in the window, no file is written
(`../references/execution/cli.md` § Download the traces).

**Serving only: delete the deployment** when the student should stop answering:
`distil deployment delete <deployment-id>`. After it, `deployment_status` stays `JOB_SUCCESS` and
`endpoint_status` reads `stopped`; the endpoint answers from the fallback. Note it in `run.md`.

## Step 5: Read the output

Read one record before converting. Each holds the request and the response as JSON strings under
`input` and `output`, and `metadata.status` (the HTTP status the caller got) and
`metadata.source` (`primary` or `fallback`). Failed calls are recorded too.

- The records hold the production model's or the student's answers, not verified labels: they
  become rows only through relabelling.
- Once a student sits behind the endpoint, records after its deployment stopped are the
  fallback's answers. Filter on `metadata.source` when only one model's answers are wanted.
- Record the count of usable records (status 200, non-empty reply) in the run log.
