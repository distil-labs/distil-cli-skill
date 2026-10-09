# Inference Endpoints

How an inference endpoint behaves. Procedure: `../stages/inference-endpoint.md`. Commands:
`execution/cli.md` § Inference endpoints.

## What an endpoint is

An inference endpoint is an OpenAI-compatible gateway at `https://inference.distillabs.ai/v1`.
An application calls it with an ordinary chat completions request, and the endpoint keeps a
record of every call. It has two models behind it:

- **The fallback**: the model the user runs in production today, named by its OpenRouter slug
  in `owner/model` form (https://openrouter.ai/models lists them). Every endpoint has one.
- **The primary**: in this skill, a hosted deployment of a trained SLM. An endpoint has either
  none or one.

| | Collecting endpoint | Serving endpoint |
|---|---|---|
| Primary | none | a hosted deployment of the trained SLM |
| Who answers | the fallback, always | the SLM; the fallback when the SLM fails or has stopped |
| What its records are for | the traces the first model is built from | the traces of the next iteration |

A request goes to the primary first. When the primary fails, a timeout included, or there is
no primary, the fallback answers. The request is forwarded as received, with `model` rewritten
to the name the model behind it serves, so the application never sees which one answered.

## Names

The name given at creation is a prefix. The platform appends a random suffix and returns the
**unique endpoint name** (`support` → `support-yeOdAS`). Every command takes the unique name,
and the `model` field of every request carries it. There is no lookup by prefix and no rename;
a lost name is recovered from the endpoint list.

## Lifetime

An endpoint is permanent. It has no idle timeout, and it cannot be edited or deleted. Changing
the fallback or the primary means a new endpoint with a new unique name, and the application
moves its `model` string to it.

The deployment behind a primary is a session for trying the model on real traffic: it stops
after six hours, or after one hour with no traffic, and it cannot be restarted. From then on the
endpoint answers every call from the fallback. Serving the student again needs a new deployment
and a new endpoint in front of it. A running deployment bills until it stops, so delete it when
it is no longer needed; after a delete the endpoint answers from the fallback. To serve a model
permanently, the customer contacts contact@distillabs.ai.

The deployment serves the SLM's LoRA adapter with vLLM on the student's HuggingFace base model
(`deployment.md` § Artifacts). An SLM trained with a `tuning.lora_r` that is not one of 8, 16, 32,
64, 128, 256, 320 or 512 cannot be deployed.

To test an existing SLM on a new test set, deploy it, send the test rows through
`model_client.py` pointed at the endpoint, and compare the answers with the references. No
command scores an existing SLM on new rows.

## Keys

A request authenticates with an inference API key, which is a different credential from the
CLI session. A key authenticates nothing until it is linked to an endpoint. The link is
many-to-many, so one key can serve the collecting endpoint and the serving endpoint after it.
An account holds at most 5 keys. The secret is shown once, at creation.

A key change takes up to a minute to reach the endpoint. A freshly linked key can be rejected
during that minute: wait and retry rather than reporting a failure.

## Records

The endpoint records the fraction of calls set by the trace sample rate at creation. The CLI
sends 1 (every call) unless `--trace-sample-rate` says otherwise; an API create that omits the
field records one call in a hundred. The rate cannot be changed afterwards.

A record holds the request and the response as JSON strings under `input` and `output`, and
`metadata` with the HTTP `status` the caller received and the `source`: `fallback` or
`primary`. Failed calls are recorded too. The record layout and the conversion to a trace
file: `data-preparation/traces.md` § From an inference endpoint.

## From records to a traces object

There are two ways to turn an endpoint's records into a PreparedTraces:

| | Directly from the endpoint | Download, convert, upload |
|---|---|---|
| How | the platform reads the endpoint's records itself | download the records, convert them to `{"messages": [...]}`, upload the file |
| Delay | a call becomes available about 24 hours after it was made | none: a call can be downloaded right after it was made |
| Size | no limit: every record the endpoint holds | a few thousand traces at most |
| Filtering | every record goes in; relabelling skips failed calls | you choose: by status, by `source`, by time window |
| Next step | `dataset create-from-traces` with a config (`observation_format: langfuse`) and a job description | `dataset create --traces` with the config, the job description and any test set to carry |

Use the direct route for the full trace set. Use the download route when the traces are needed
now, when there are only a few thousand, when the records must be filtered first, for example
on `source` once a student sits in front of the fallback, or when the current `test.jsonl` has
to go into the same Dataset (`../workflows/build-a-model.md` Step 10).

## Credits

Creating an endpoint spends `inference_endpoints_post`, a deployment `deployments_from_slms_post`
(again for every replacement of a stopped deployment), a traces object by the direct route
`prepared_traces_post`, and the Dataset after it `datasets_from_prepared_traces_post` (or
`datasets_post` for the download route). Balances: `platform.md` § Credits.
