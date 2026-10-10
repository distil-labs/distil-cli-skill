# Execution Backend: distil labs API

How stages run on the platform. Stage files link here by operation name. Every stage is an
entity created through the REST API, and every entity is created either by staging files or
by running a job over the entity before it. The prerequisites are a distil labs account and the
`distil` CLI signed in to it, because the CLI mints the access tokens. There is no API route that
creates an account: sign up with `distil signup`.

`cli.md` is the default backend and this one is the alternative. Take it when the user
prefers it, or when the work is already scripted in Python. `README.md` § Choose the backend
decides between them, and the choice goes in `run.md`.

## Prerequisites

```bash
pip install requests pyyaml
distil whoami                                        # prints the current user
```

Install the CLI and sign in per `README.md` § Set up the CLI. The API is
`https://api.distillabs.ai` and takes a bearer token from `distil access-token`, which refreshes
the CLI session and prints a new access token each time it runs. Override the URL for
development, with a CLI signed in to the same environment:

```bash
export DL_PLATFORM_URL=https://api-dev.distillabs.ai
```

Access tokens are short-lived, so the preamble below fetches a token per request rather than
holding one. `pyyaml` is needed because a config
is read back as `config.yaml` text and sent as a JSON object.

## Preamble

Every snippet in this file assumes this block.

```python
import json
import os
import subprocess
import time
from pathlib import Path

import requests
import yaml

PLATFORM_URL = os.getenv("DL_PLATFORM_URL", "https://api.distillabs.ai")
POLL_INTERVAL_SECONDS = 20
POLL_TIMEOUT_SECONDS = 6 * 60 * 60

# A staging response names each file by extension; a create body names it by
# field. These maps are the only place that difference is spelled out.
DATASET_FIELDS = {
    "config": "config_yaml",
    "job_description": "job_description_json",
    "train_data": "train_data_jsonl",
    "test_data": "test_data_jsonl",
    "traces": "traces_jsonl",
}
PREPARED_TRACES_FIELDS = {"traces_jsonl": "traces_jsonl"}
SLM_FIELDS = {"model": "model_tar", "config": "config_yaml"}


def auth():
    """Tokens are short-lived, so this is called per request rather than cached. A failure carries the CLI's message,
    such as `Not logged in or session expired`."""
    result = subprocess.run(
        ["distil", "access-token"], capture_output=True, text=True
    )
    if result.returncode != 0:
        raise RuntimeError(result.stderr.strip())
    return {"Authorization": f"Bearer {result.stdout.strip()}"}


def raise_with_body(response):
    """raise_for_status hides the response body, which is where validation
    errors live. Print it before raising."""
    try:
        response.raise_for_status()
    except requests.exceptions.HTTPError:
        print(f"{response.status_code}: {response.text[:2000]}")
        raise


def get(path):
    response = requests.get(f"{PLATFORM_URL}{path}", headers=auth())
    raise_with_body(response)
    return response.json()


def post(path, body):
    response = requests.post(
        f"{PLATFORM_URL}{path}",
        data=json.dumps(body),
        headers={"content-type": "application/json", **auth()},
    )
    raise_with_body(response)
    return response.json()


def stage(staging_route, files, fields):
    """PUT each local file to its presigned URL and return the create body.
    Only the files given are staged and named: a Dataset file left out is an
    empty split."""
    urls = get(staging_route)
    body = {}
    for field, filepath in files.items():
        url = urls[fields[field]]
        put = requests.put(url, data=Path(filepath).read_bytes())
        put.raise_for_status()
        body[field] = url
    return body


def metadata_url(collection, entity_id, field):
    """Presigned URL for a parent's config or job description.

    For a Dataset an expand produced, download-metadata fills these only
    once its job has finished; while it runs, both are null. Raise something that names the
    entity and its status, because passing the null on to requests.get
    fails with "Invalid URL 'None': No scheme supplied", which names
    neither and sends you hunting for a bug in your URL building.
    """
    url = get(f"/{collection}/{entity_id}/download-metadata")[field]
    if url is None:
        status = get(f"/{collection}/{entity_id}/status")["status"]
        raise RuntimeError(
            f"{collection}/{entity_id}: {field} is not available yet "
            f"(status {status}). Wait for JOB_SUCCESS before reading it."
        )
    return url


def config_of(collection, entity_id):
    """Fetch a parent's whole config as a dict.

    An override replaces the config rather than merging into it, so every
    override starts here: fetch, edit one field, send the whole thing back.
    This read costs no credits.
    """
    response = requests.get(metadata_url(collection, entity_id, "config_url"))
    response.raise_for_status()
    return yaml.safe_load(response.text)


def job_description_of(collection, entity_id):
    """Fetch a parent's whole job description as a dict. Same rule as
    config_of: it is replaced wholesale, not merged."""
    response = requests.get(
        metadata_url(collection, entity_id, "job_description_url")
    )
    response.raise_for_status()
    return response.json()


def credits():
    """Remaining calls per metered route, keyed by route name. Free, and
    answers at zero balance.

    Only metered routes appear. A route the platform never charges for is
    absent from the dict rather than reported as unlimited, so read this
    with .get() unless you know the route is metered.
    """
    return get("/credits/endpoints")["balances"]


def poll(collection, entity_id, timeout_seconds, status_field="status"):
    """Block until the job reaches a final state. Raises on failure with the
    tail of the job log, which is where the cause is. A Dataset that no job
    produced answers JOB_SUCCESS at once."""
    deadline = time.time() + timeout_seconds
    while time.time() < deadline:
        status = get(f"/{collection}/{entity_id}/status")[status_field]
        if status == "JOB_SUCCESS":
            return
        if status in ("JOB_FAILURE", "JOB_STOPPED"):
            logs = get(f"/{collection}/{entity_id}/logs")["logs"]
            raise RuntimeError(f"{collection}/{entity_id} {status}:\n{logs[-4000:]}")
        print(f"{collection}/{entity_id}: {status}", flush=True)
        time.sleep(POLL_INTERVAL_SECONDS)
    raise TimeoutError(
        f"{collection}/{entity_id} did not finish within {timeout_seconds}s"
    )
```

## The entity model

The routes for each stage. What the entities are and how they chain: `../platform.md`
§ Entities and jobs. The `Override` column says whether a job's configuration can be varied at
submission. See § Overrides.

| Stage | Entity | Submit | Read | Override |
|---|---|---|---|---|
| (job input) | PreparedTraces | `POST /prepared-traces`, from a staged `traces.jsonl` or an endpoint's name | `GET /prepared-traces[/<id>]`, `GET /prepared-traces/<id>/download` | no config of its own |
| (job input) | Dataset | `POST /datasets` from staged files, or `POST /datasets/from-prepared-traces` with a config and a job description inline | `GET /datasets[/<id>]`, `GET /datasets/<id>/{status,logs,metrics,sample,download,download-metadata}` | staged files only |
| relabel-traces | Dataset → Dataset | `POST /datasets/from-datasets[-smoke]` with `operation: relabel_traces_{train,test}` | same | yes |
| synthetic-data-generation | Dataset → Dataset | `POST /datasets/from-datasets[-smoke]` with `operation: generate_synthetic_data_{train,test}` | same | yes |
| teacher-evaluation | TeacherEvaluation | `POST /teacher-evaluations` with `from_dataset_id` | `GET /teacher-evaluations[/<id>]`, `GET /teacher-evaluations/<id>/{status,logs,metrics,download-metadata}` | yes |
| model-training | SLM | `POST /slms/from-datasets[-smoke]`, or `POST /slms` from a staged tarball | `GET /slms[/<id>]`, `GET /slms/<id>/{status,logs,metrics,download,download-metadata}` | yes |
| inference-endpoint | Deployment, InferenceEndpoint | `POST /deployments/from-slms`, then `POST /inference-endpoints` with the deployment as `primary`; or `POST /inference-endpoints` with no primary | `GET /deployments/<id>/{status,endpoint,logs}`, `DELETE /deployments/<id>`, `GET /inference-endpoints[/<name>]` | no config of its own |

Seed datasets, training datasets and 410 responses: `../migrating-old-entities.md`.

### Supplying files

Staging files is a three-step exchange: `GET /staging-<kind>-s3-urls` returns a presigned PUT
URL per file, each file is PUT to its URL, and those URLs are posted back as the create body.
`stage()` in the preamble does all three. The staging response names a file by extension
(`train_data_jsonl`) while the create body names it by field (`train_data`), and the maps in
the preamble are the only place that mapping is spelled out.

`download-metadata` returns presigned URLs for an entity's `config.yaml` and
`job_description.json` at no credit cost. `GET /slms/<id>/download-metadata` returns a third
field, `model_client_url`. A PreparedTraces has no config or job description.

## Credits

`GET /credits/endpoints` reports the calls remaining on each metered route, keyed by route
name. The read is free and answers at zero balance. How metering works, the route table and
the starting balances: `../platform.md` § Credits.

```python
balances = credits()
print(balances["datasets_from_datasets_generate_synthetic_data_post"])
```

Routes the platform never charges for are absent from the response, so read an unfamiliar key
with `.get()`. A submission against an exhausted route fails 402. A `-smoke` route spends its
own `*_smoke_post` key, never the full run's. `POST /datasets/from-datasets` is metered by the
operation in the body: `datasets_from_datasets_relabel_traces_post` for the two relabelling
operations, `datasets_from_datasets_generate_synthetic_data_post` for the two generation ones.

## Creating the job inputs

### The traces object

```python
body = stage("/staging-prepared-traces-s3-urls", {"traces_jsonl": "traces.jsonl"}, PREPARED_TRACES_FIELDS)
prepared_traces_id = post("/prepared-traces", body)["id"]
```

The create checks only that the file is there. A malformed trace fails the first expand that
reads it, and the job log names the cause. An endpoint's records go in the same route with
`inference_endpoint_name` in place of `traces_jsonl` (§ Traces from an endpoint).

### The Dataset

From a traces object, with the config and the job description inline as JSON objects:

```python
dataset_id = post("/datasets/from-prepared-traces", {
    "from": prepared_traces_id,
    "config": yaml.safe_load(Path("config.yaml").read_text()),
    "job_description": json.loads(Path("job_description.json").read_text()),
})["id"]
```

No job runs: the traces are copied, train and test are empty, and the Dataset is ready at
once. The trace file's shape must match `trace_processing.observation_format` in the config
(`../data-preparation/traces.md`).

From files on disk (`../data-preparation/overview.md`), staged file by file. `config` and
`job_description` are required; stage any of `train_data`, `test_data` and `traces`, and leave
out the ones the user does not have:

```python
body = stage(
    "/staging-datasets-s3-urls",
    {
        "config": "config.yaml",
        "job_description": "job_description.json",
        "train_data": "train.jsonl",
        "test_data": "test.jsonl",
        # "traces": "traces.jsonl",
    },
    DATASET_FIELDS,
)
dataset_id = post("/datasets", body)["id"]
```

`POST /datasets` validates the bundle and returns 400 with the validation error when it fails.
That is this backend's dryrun, and `raise_with_body` is what makes the message visible instead
of a bare `HTTPError`. The route is metered (`datasets_post`), but only a successful create
spends a credit: the balance is checked first, and the call is recorded only after validation
passes. So validating a broken bundle repeatedly is free, and a 402 here means the balance was
already zero before the bundle was ever read.

### The SLM from staged files

An SLM is normally *produced* by training. An existing one, such as a `model.tar` and
`config.yaml` downloaded from another SLM, is registered by staging the two files. This trains
nothing:

```python
body = stage("/staging-slms-s3-urls", {"model": "model.tar", "config": "config.yaml"}, SLM_FIELDS)
slm_id = post("/slms", body)["id"]
poll("slms", slm_id, POLL_TIMEOUT_SECONDS)
```

The tarball must hold the LoRA adapter under `model-adapter/`, as `GET /slms/<id>/download`
produces it (`../deployment.md` § Artifacts); the config travels beside it, not inside. The
platform expands the tarball into the layout a trained SLM has, which is the job `poll()`
waits on. Files that do not form a valid SLM answer 400. The route is metered on `slms_post`,
which starts at zero, so it needs a grant before the first upload (`../platform.md` § Credits).

## Submitting jobs

Every job is created by posting the id of the entity before it. Nothing is uploaded at this
point, because the parent's files are already on the platform.

### Relabel traces

One route runs all four expand operations, relabelling and generation alike (`../platform.md` § The expand operations). The body names the parent
and the operation; the new Dataset's `parent_dataset_id` is the parent and its `operation` is
the one sent:

```python
test_relabelled_id = post("/datasets/from-datasets", {
    "from": dataset_id, "operation": "relabel_traces_test",
})["id"]
poll("datasets", test_relabelled_id, POLL_TIMEOUT_SECONDS)

train_relabelled_id = post("/datasets/from-datasets", {
    "from": test_relabelled_id, "operation": "relabel_traces_train",
})["id"]
poll("datasets", train_relabelled_id, POLL_TIMEOUT_SECONDS)

train_generated_id = post("/datasets/from-datasets", {
    "from": train_relabelled_id, "operation": "generate_synthetic_data_train",
})["id"]
poll("datasets", train_generated_id, POLL_TIMEOUT_SECONDS)
```

The count comes from the parent's config, `trace_processing.num_{split}_relabelled`, or from
the `config` override. A smoke is the same body posted to `/datasets/from-datasets-smoke`,
which forces the count to 128 whatever the config says (`../platform.md` § Smoke runs) and
spends `datasets_from_datasets_smoke_post`.

### Synthetic data generation

The same route and body with `operation: generate_synthetic_data_train` or
`generate_synthetic_data_test`; the target is `synthgen.{split}_generation_target`, and the
`-smoke` route forces it to 128:

```python
smoke_id = post("/datasets/from-datasets-smoke", {
    "from": dataset_id, "operation": "generate_synthetic_data_train", "config": config,
})["id"]
poll("datasets", smoke_id, POLL_TIMEOUT_SECONDS)
```

The smoke's Dataset is a side branch; the full run is posted on the same parent. Errors: 400 for
a config that fails validation, 404 for a missing parent, 409 for a parent whose job has not
succeeded, 402 for insufficient credits. A job that fails validation of the Dataset it produced
(for example a classification split missing a class) ends in `JOB_FAILURE` with the cause in the
log.

### Overrides

Every job create accepts optional `config` and `job_description` objects inline, alongside
`from`: `POST /datasets/from-datasets` and `-smoke`, `POST /teacher-evaluations` (alongside
`from_dataset_id`), `POST /slms/from-datasets` and `-smoke`. The two are independent, and there
is no staging step for either. An omitted one is read from the parent Dataset's own files.

Each replaces the parent's file whole, and anything you leave out reverts to a library default
(`../platform.md` § Overrides). So an override is read-edit-resend, never hand-built:

```
config_of("datasets", id)  →  edit one field  →  POST it whole
```

`config_of()` and `job_description_of()` do the read. Two shapes to know:

- A config naming only one field under `base`, such as
  `{"config": {"base": {"student_model_name": …}}}` (the shape most people guess), is a 400.
  `base` is required and has no default, so that is not a config.
- A config carrying a valid `base` but omitting `synthgen` or `tuning` is *accepted*, and those
  sections take defaults. Dropping one field from a section you do send reverts that field the
  same way. Nothing errors.

Check what you are about to send against what you read, before you spend anything:

```python
parent = config_of("datasets", dataset_id)
config = json.loads(json.dumps(parent))          # a copy to edit
config.setdefault("synthgen", {})["train_generation_target"] = 512

dropped = {
    f"{section}.{key}"
    for section, values in parent.items() if isinstance(values, dict)
    for key in values if key not in config.get(section, {})
}
assert not dropped, f"these would revert to defaults: {dropped}"
```

An expand that ran with an override writes the overridden files into the Dataset it produces,
so `config_of("datasets", child_id)` reads them back.

### Teacher evaluation

```python
teacher_evaluation_id = post("/teacher-evaluations", {"from_dataset_id": dataset_id})["id"]
poll("teacher-evaluations", teacher_evaluation_id, POLL_TIMEOUT_SECONDS)
```

Another teacher, or sharper judge instructions, is the same Dataset with an override:

```python
config = config_of("datasets", dataset_id)
config["base"]["teacher_model_name"] = "<production-model>"

baseline_id = post("/teacher-evaluations", {
    "from_dataset_id": dataset_id, "config": config,
})["id"]
poll("teacher-evaluations", baseline_id, POLL_TIMEOUT_SECONDS)
```

### Model training

Training reads its config from the Dataset. Setting sensible `tuning` values before the
expands means the common case needs no override:

```python
slm_id = post("/slms/from-datasets", {"from": dataset_id})["id"]
poll("slms", slm_id, POLL_TIMEOUT_SECONDS)
```

A memory check before the full run is the same submission on `/slms/from-datasets-smoke`: one
epoch on the 128 longest train rows and the 32 longest test rows (`../platform.md` § Smoke
runs). Send it the config the full run will use, since the student and the batch size are what
it checks:

```python
config = config_of("datasets", dataset_id)
config["base"]["student_model_name"] = "<student-a>"

smoke_id = post("/slms/from-datasets-smoke", {"from": dataset_id, "config": config})["id"]
poll("slms", smoke_id, POLL_TIMEOUT_SECONDS)
```

A sweep is N submissions against the same Dataset, one per student. Read the Dataset's config
once, then vary one field per submission:

```python
base_config = config_of("datasets", dataset_id)
students = ["<student-a>", "<student-b>"]

slm_ids = {}
for student in students:
    config = json.loads(json.dumps(base_config))  # a copy per submission
    config["base"]["student_model_name"] = student
    slm_ids[student] = post("/slms/from-datasets", {"from": dataset_id, "config": config})["id"]

for student, slm_id in slm_ids.items():
    poll("slms", slm_id, POLL_TIMEOUT_SECONDS)
```

The deep copy matters: mutating one dict across iterations would send every student the last
one's value. `per_device_train_batch_size`, `memory_optimized_training` and `use_qlora` vary
the same way (read, edit, resend), which is how OOM is handled without touching the data.
Submissions run concurrently, so poll them after they are all in.

## Monitor

Poll `GET /<collection>/<id>/status` every 20 seconds. `poll()` in the preamble does this. The
status values and how to run a poller: `../platform.md` § Job status.

```python
poll("datasets", train_generated_id, POLL_TIMEOUT_SECONDS)
```

Deployments report `deployment_status` rather than `status`, so they need
`poll(..., status_field="deployment_status")`.

On failure, `GET /<collection>/<id>/logs` returns the job log, and `poll()` raises with its
tail attached.

`GET /<collection>` lists every entity in a collection, newest first, and recovers a lost id.
Each item has `id`, `created_at` and the parent's id under its own key (`dataset_id`, `slm_id`).
A Dataset also carries its `operation` and either `parent_dataset_id` or
`parent_prepared_traces_id`; a PreparedTraces its `source` (`direct_upload` or
`inference_endpoint`). The list carries no status; `GET /<collection>/<id>` returns the same
fields with `status` added (`JOB_SUCCESS` at once for a Dataset no job produced). Walking `parent_dataset_id` from a child recovers the whole chain.

```python
slms = get("/slms")
from_dataset = [slm["id"] for slm in slms if slm.get("dataset_id") == dataset_id]
```

## Output layout

Nothing is fetched by object-storage path. Outputs arrive either as JSON in the response or as
presigned URLs. A `/download` route returns one URL per file; a Dataset returns all five, an
empty split as an empty file. Which outputs each entity produces, and the conditions on them:
`../platform.md` § What each entity produces.

| Entity | `/metrics` | `/download` | `/download-metadata` | Other |
|---|---|---|---|---|
| PreparedTraces | none | `traces_url` | none | none |
| Dataset | `train_data_size_bytes`, `test_data_size_bytes`, `traces_size_bytes` | `train_data_url`, `test_data_url`, `traces_url`, `config_url`, `job_description_url`, free | free | `/sample`: up to 128 train rows |
| TeacherEvaluation | `teacher_performance`, `predictions_download_url` | none | free | none |
| SLM | `base_model_performance`, `tuned_model_performance`, `predictions_download_url` | `model_url`, `config_url` | free; also `model_client_url` | none |
| Deployment | none | none | none | none |

A field that is not ready yet reads `null`. `config_of()` turns that null into an error naming
the entity and its status, because the raw failure (`Invalid URL 'None'`) names neither.

### Reading a Dataset

```python
download = get(f"/datasets/{dataset_id}/download")
for field, name in [("train_data_url", "train.jsonl"), ("test_data_url", "test.jsonl"), ("traces_url", "traces.jsonl")]:
    Path(name).write_bytes(requests.get(download[field]).content)
```

An empty split is an empty file. The rows of a split are the parent's rows followed by the ones
the expand added, and nothing in a row says which; download the parent too and diff to isolate
the new rows. `/sample` returns `{"rows": [...]}`, up to 128 rows of the train split, for a
quick look without the download.

## Fetch metrics

Aggregated metrics arrive as the `*_performance` object in a `/metrics` response: a
metric-name-to-score dict, flat for every task except classification (below). The per-example
detail is behind the presigned `predictions_download_url` alongside it.

```python
metrics = get(f"/teacher-evaluations/{teacher_evaluation_id}/metrics")
print(metrics["teacher_performance"])

slm_metrics = get(f"/slms/{slm_id}/metrics")
print(slm_metrics["base_model_performance"], slm_metrics["tuned_model_performance"])

predictions = requests.get(slm_metrics["predictions_download_url"]).text
```

The production model's score is the `teacher_performance` of the teacher evaluation that ran
with it as the teacher (`../../stages/teacher-evaluation.md`).

Two things about the predictions file:

- It is JSONL, one test example per line, with `prompt`, `completion`, `prediction` and
  that example's own scores. Read a row and work from what is there.
- For classification the performance object is not flat. Alongside `accuracy` it carries one
  key per class label, each holding `{precision, recall, f1-score, support}`, which gives
  per-class precision and recall for free. There is no `confusion_matrix` and no
  `classification_report`. Iterate by key rather than assuming numeric values: `accuracy` is
  a float and every other entry is a dict.

  ```python
  performance = get(f"/slms/{slm_id}/metrics")["tuned_model_performance"]
  accuracy = performance["accuracy"]
  per_class = {k: v for k, v in performance.items() if isinstance(v, dict)}
  ```

## Fetch model artifacts

```python
download = get(f"/slms/{slm_id}/download")
Path("model.tar").write_bytes(requests.get(download["model_url"]).content)
Path("config.yaml").write_bytes(requests.get(download["config_url"]).content)
```

The tarball expands to the layout `../deployment.md` § Artifacts describes.

### The inference client on its own

`download-metadata` presigns `model_client.py` separately from the tarball, a few kilobytes
rather than several gigabytes. This is the only `download-metadata` response with three fields.
Every other entity returns `config_url` and `job_description_url` alone.

```python
client_url = metadata_url("slms", slm_id, "model_client_url")
Path("model_client.py").write_text(requests.get(client_url).text)
```

`model_client_url` is null until the SLM reaches `JOB_SUCCESS`, and `metadata_url()` turns that
null into an error naming the entity and its status. It misreads one case: an SLM created by
`POST /slms` whose tarball carried no client is null permanently, and the helper still says to
wait. When the status is already `JOB_SUCCESS`, the file is not there to wait for.

What the client is for and how to call it: `../deployment.md`.

## Reuse the data for training-only runs

Retrain the same Dataset by id under a config override. Nothing is downloaded, copied or
reassembled:

```python
config = config_of("datasets", dataset_id)
config.setdefault("tuning", {})["num_train_epochs"] = 6

retrained = post("/slms/from-datasets", {"from": dataset_id, "config": config})["id"]
```

This is the same call as the sweep in § Submitting jobs, with one field changed instead of
the student.

### Deploy an SLM

```python
deployment_id = post("/deployments/from-slms", {"from": slm_id})["id"]
poll("deployments", deployment_id, POLL_TIMEOUT_SECONDS, status_field="deployment_status")
```

A deployment is never called directly. Once it reaches `JOB_SUCCESS`, an inference endpoint with
the deployment id as its `primary` serves it (§ Serve the student behind an endpoint). Query that
endpoint through the model's own client, not a hand-built request: `../deployment.md` § Serving
hosted has the call and says why.

```python
requests.delete(f"{PLATFORM_URL}/deployments/{deployment_id}", headers=auth())
```

The job does not return until vLLM answers, so `JOB_SUCCESS` means serving rather than merely
scheduled. After the delete, `deployment_status` stays `JOB_SUCCESS`. `endpoint_status` going
to `stopped` is what confirms the deployment is down. Delete the deployment when finished and
check that field, because a running deployment bills until its idle timeout.

## Inference endpoints

An inference endpoint is an OpenAI-compatible gateway in front of the model the user already runs
in production. Their application calls the endpoint instead of the provider, the fallback model
answers as before, and the platform keeps a copy of every call. Those copies are the traces the
Dataset is built from, so this is the route for a user who wants a distilled model but has no
trace file to start from.

This is not a Deployment. It is permanent, has no idle timeout, cannot be edited or deleted, and
serves no trained model of its own until one is set as its primary (§ Serve the student behind an
endpoint).

### Create an endpoint

```python
endpoint = post("/inference-endpoints", {
    "name-prefix": "support",
    "fallback": {"model": "openai/gpt-4.1-mini"},
    "trace-sample-rate": 1,
})
name = endpoint["unique_endpoint_name"]     # "support-yeOdAS"
```

`name-prefix` is a prefix, not the name. The platform appends a suffix and returns
`unique_endpoint_name`, which every other route takes and which a request body carries. Record it
in `run.md`. `get(f"/inference-endpoints/{name}")` reads one back, and
`get("/inference-endpoints")` lists them newest first, which is how a lost name is recovered.

`fallback.model` takes an OpenRouter model slug in `owner/model` form
(https://openrouter.ai/models lists every slug it accepts). It must name the model the user
already calls in production, so ask rather than guess. If they are undecided, these are the ones
distil labs runs today: `openai/gpt-4.1-mini` (small, cheap, the most common), `openai/gpt-5.4`,
`google/gemini-2.5-flash`, `google/gemini-3.1-flash-lite`.

`trace-sample-rate` is the fraction of calls the endpoint records, 0 to 1. Send 1 unless the
traffic is very high. An endpoint created without the field records one call in a hundred; the
response's `trace_sampling_rate` says what an existing one does.

`POST /inference-endpoints` is metered (`inference_endpoints_post`) and answers 402 once that
balance is spent.

### Keys

```python
key = post("/api-keys", {"api-key-name": "support-prod"})
secret = key["secret"]      # returned by this response and by nothing else

requests.put(
    f"{PLATFORM_URL}/inference-endpoints/{name}/api-keys/support-prod", headers=auth()
).raise_for_status()
```

`GET /api-keys` lists names and creation dates, never secrets, and nothing reissues one. Tell the
user where the secret is going before creating it, and never echo it into the transcript or
`run.md`. `DELETE` on the link path unlinks; `DELETE /api-keys/<name>` revokes the key
everywhere.

Key changes take up to a minute to propagate. A 409 from either link route means the endpoint's
datastore has not caught up with a write moments earlier, and a link that answered 204 can take a
moment longer before the endpoint honours the key. Wait and retry rather than reporting a
failure, and do not have the user move traffic onto a new key, or revoke the key it replaces,
inside that minute.

### The call the application makes

The endpoint is a different host from `PLATFORM_URL` and takes the key's secret, not an access
token:

```python
requests.post(
    "https://inference.distillabs.ai/v1/chat/completions",
    headers={"Authorization": f"Bearer {secret}", "Content-Type": "application/json"},
    json={"model": name, "messages": [{"role": "user", "content": "Say hi."}]},
)
```

`model` carries the unique endpoint name rather than a model name. Everything else is an ordinary
chat completions request, so an existing OpenAI client changes three strings: base URL, key and
model. Send one call like this with the user before they touch their application. Then it waits:
traces accumulate at the rate of their traffic, and the test set and the train set want hundreds,
so the next stage is days or weeks away rather than minutes. Say that plainly when proposing this
route.

### Read the traces

```python
def traces(endpoint_name, limit=None, **window):
    """Yield an endpoint's traces, newest first, a page at a time."""
    cursor, yielded, seen = None, 0, set()
    while True:
        params = dict(window, **({"pagination-cursor": cursor} if cursor else {}))
        response = requests.get(
            f"{PLATFORM_URL}/inference-endpoints/{endpoint_name}/traces",
            params=params,
            headers=auth(),
        )
        raise_with_body(response)
        page = response.json()
        for trace in page["traces"]:
            yield trace
            yielded += 1
            if limit is not None and yielded >= limit:
                return
        cursor = page["pagination_cursor"]
        if cursor is None or cursor in seen:
            return
        seen.add(cursor)
```

The page size is the platform's, so `limit` caps what is kept and not what is transferred.
Terminate on `pagination_cursor` being `null` and on nothing else, because an empty page can
still carry a cursor; the `seen` guard stops a repeated cursor looping forever.

`from-start-time` and `to-start-time` set the window as ISO 8601 timestamps. Always send both:

```python
recent = list(traces(
    name,
    limit=1000,
    **{"from-start-time": "2026-09-01T00:00:00Z", "to-start-time": "2026-10-01T00:00:00Z"},
))
```

**What comes back is not a trace file.** It is the platform's record of each call:
identifiers, timings and metadata around the request and the response. Relabelling wants one
`{"messages": [...]}` object per line. Inspect a record before writing any conversion, convert per
`../data-preparation/traces.md` § From an inference endpoint, then stage the result as
`traces_jsonl` of a traces object or as `traces` of a Dataset like any other trace file.

Each record holds the request and the response as JSON strings under `input` and `output`, and
`metadata` carries the HTTP `status` and `source`, which names whether the `fallback` or the
`primary` answered.

### Export the traces to a file

A traces export writes the endpoint's records in a window to one file, rather than pages:

```python
export = post(f"/inference-endpoints/{name}/traces-exports", {
    "from-start-time": "2026-09-01T00:00:00Z",
    "to-start-time": "2026-10-01T00:00:00Z",
})
Path("raw-traces.jsonl").write_bytes(requests.get(export["download_url"]).content)
```

The create answers 201 with `id`, `created_at`, the window it used and a presigned
`download_url`. The window follows the same rule as `/traces`: send both bounds. Earlier exports
are listed by `GET /inference-endpoints/<name>/traces-exports` and read back one at a time by
`GET /inference-endpoints/<name>/traces-exports/<export-id>`, both with the same fields. Read a
record of the file before writing a conversion, as with `/traces`: it is not a trace file until
converted.

### Traces from an endpoint

`POST /prepared-traces` also takes the endpoint's records directly, with nothing downloaded,
converted or staged:

```python
prepared_traces_id = post("/prepared-traces", {"inference_endpoint_name": name})["id"]
```

Send `traces_jsonl` or `inference_endpoint_name`, not both. The Dataset created from it
(§ The Dataset) needs `trace_processing.observation_format: langfuse` in its config, since the
records are not `{"messages": [...]}` lines. Which records go in, how fresh they are, and when
to take this route over the download: `../inference-endpoints.md` § From records to a traces
object.

### Serve the student behind an endpoint

The same route puts a trained model in front of the traffic. Deploy the student (§ Deploy an
SLM), wait for `JOB_SUCCESS`, then create a new endpoint with the deployment id as
`primary`:

```python
endpoint = post("/inference-endpoints", {
    "name-prefix": "support-slm",
    "fallback": {"model": "openai/gpt-4.1-mini"},
    "primary": deployment_id,
    "trace-sample-rate": 1,
})
```

The route answers 409 until the deployment has reached `JOB_SUCCESS`. Smoke-test a few test-set
rows through `model_client.py` pointed at the endpoint before the application moves, and confirm
in the downloaded records that `source` reads `primary` for them.

For a server the user runs themselves, `primary` is an object instead:
`{"url": "https://<server>", "api-key": "<key>"}`, the base URL without `/v1`, since the endpoint
appends `/v1/chat/completions` itself; both fields or neither.
`primary.readiness-gate-timeout-ms` is optional and for internal use. The endpoint calls the primary first and falls back to `fallback.model` whenever
the primary fails, a timeout included. An endpoint cannot be edited, so this is always a new
endpoint with a new unique name, and the application's `model` string moves to it. The hosted
deployment stops on its own, after which every call goes to the fallback and `source` reads
`fallback`; a permanent primary is a request to contact@distillabs.ai. The new endpoint records
like the first one, and its records are the traces for the next iteration.
