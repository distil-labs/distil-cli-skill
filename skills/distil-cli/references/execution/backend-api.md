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

Access tokens are short-lived, much shorter than a synthgen or training run, so the preamble
below fetches a token per request rather than holding one. `pyyaml` is needed because a config
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

# A staging response names each file by extension; a create body names it by
# field. These maps are the only place that difference is spelled out.
STAGING_FIELDS = {
    "train_data": "train_data_jsonl",
    "test_data": "test_data_jsonl",
    "config": "config_yaml",
    "job_description": "job_description_json",
    "unstructured_data": "unstructured_data_jsonl",
    "model": "model_tar",
}
PREPARED_TRACES_FIELDS = {
    "traces_jsonl": "traces_jsonl",
    "config": "config_yaml",
    "job_description_json": "job_description_json",
    "test_jsonl": "test_jsonl",
}


def auth():
    """Tokens are short-lived, much shorter than a training run, so this is
    called per request rather than cached. A failure carries the CLI's message,
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


def stage(staging_route, files, fields=STAGING_FIELDS):
    """PUT each local file to its presigned URL and return the create body."""
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

    download-metadata fills these only once the entity's own job has
    finished; while it runs, both are null. Raise something that names the
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
    tail of the job log, which is where the cause is."""
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
| (job input) | PreparedTraces | `POST /prepared-traces`, from staged traces or an endpoint's records | `GET /prepared-traces/<id>/{status,download,download-metadata}` | staged files only |
| test-set-from-traces | PreparedTraces → PreparedTraces | `POST /prepared-traces/with-expanded-test-set` | `GET /prepared-traces/<id>/{status,logs,metrics,download,download-metadata}` | yes |
| trace-processing | PreparedTraces → SeedDataset | `POST /seed-datasets/from-prepared-traces` | `GET /seed-datasets/<id>/{status,logs,download,download-metadata}` | yes |
| (job input) | SeedDataset | `POST /seed-datasets` | `GET /seed-datasets/<id>/{status,download,download-metadata}` | staged files only |
| teacher-evaluation | TeacherEvaluation | `POST /teacher-evaluations/from-seed-datasets` | `GET /teacher-evaluations/<id>/{status,logs,metrics,download-metadata}` | yes |
| synthetic-data-generation | TrainingDataset | `POST /training-datasets/from-seed-datasets[-smoke]`, or `POST /training-datasets` from staged files | `GET /training-datasets/<id>/{status,logs,metrics,sample,download,download-metadata}` | yes |
| model-training | SLM | `POST /slms/from-training-datasets[-smoke]`, or `POST /slms` from a staged tarball | `GET /slms/<id>/{status,logs,metrics,download,download-metadata}` | yes |
| model-deployment | Deployment | `POST /deployments/from-slms` | `GET /deployments/<id>/{status,endpoint,logs}`, `DELETE /deployments/<id>` | no config of its own |

The `/uploads`, `/staging-uploads-s3-urls`, `/teacher-evaluations/from-uploads` and
`/training-datasets/from-uploads` paths answer 410 Gone naming the route to use instead, so
a 410 means the path is the problem, not the payload.

Staging files is a three-step exchange: `GET /staging-<kind>-s3-urls` returns a presigned PUT
URL per file, each file is PUT to its URL, and those URLs are posted back as the create body.
`stage()` in the preamble does all three. The staging response names a file by extension
(`train_data_jsonl`) while the create body names it by field (`train_data`), and `stage()` is
the only place that mapping is spelled out. Staged bundles expire (`../platform.md`
§ Entities and jobs).

`download-metadata` returns presigned URLs for an entity's `config.yaml` and
`job_description.json` at no credit cost. `GET /slms/<id>/download-metadata` returns a third
field, `model_client_url`. Every other entity returns exactly two.

## Credits

`GET /credits/endpoints` reports the calls remaining on each metered route, keyed by route
name. The read is free and answers at zero balance. How metering works, the route table and
the starting balances: `../platform.md` § Credits.

```python
balances = credits()
print(balances["training_datasets_from_seed_datasets_post"])
```

Routes the platform never charges for are absent from the response, so read an unfamiliar key
with `.get()`. A submission against an exhausted route fails 402. A `-smoke` route spends its
own `*_smoke_post` key, never the full run's.

## Submitting jobs

Every job is created by posting the id of the entity before it. Nothing is uploaded at this
point, because the parent's files are already on the platform.

| Stage | Collection | Typical timeout |
|---|---|---|
| Test set from traces | `prepared-traces` | scales with `num_traces_to_relabel` and `num_synthetic_examples` |
| Trace processing | `seed-datasets` | 45 min |
| Teacher evaluation | `teacher-evaluations` | 30 min |
| Synthetic data generation | `training-datasets` | 90 min |
| Model training | `slms` | 90 min |
| Deployment | `deployments` | 40 min |

### Test set from traces and trace processing

Stage the traces, build a test set from them, then process the result into a SeedDataset.

```python
body = stage(
    "/staging-prepared-traces-s3-urls",
    {
        "traces_jsonl": "traces.jsonl",
        "config": "config.yaml",
        "job_description_json": "job_description.json",
        # Add "test_jsonl" for test rows of your own. The test set from
        # traces job adds to them.
    },
    PREPARED_TRACES_FIELDS,
)
prepared_traces_id = post("/prepared-traces", body)["id"]

updated_traces_id = post(
    "/prepared-traces/with-expanded-test-set", {"from": prepared_traces_id}
)["id"]
# Raise the timeout for a large num_traces_to_relabel or num_synthetic_examples.
poll("prepared-traces", updated_traces_id, 60 * 60)

seed_dataset_id = post(
    "/seed-datasets/from-prepared-traces", {"from": updated_traces_id}
)["id"]
poll("seed-datasets", seed_dataset_id, 60 * 45)
```

`POST /prepared-traces` checks only that the files are there and that the config parses. A
malformed trace, test row or job description fails the first job that reads it, and the job
log names the cause. An uploaded PreparedTraces runs no job, so its status is `JOB_SUCCESS` as
soon as the create returns.

`with-expanded-test-set` creates a PreparedTraces whose `parent_prepared_traces_id` is the first
and whose `source` is `test_set_expansion`. Its `test.jsonl` holds the supplied test rows, the
relabelled traces and the synthetic rows; its `traces.jsonl` holds the traces the job did not
use. `from-prepared-traces` copies the PreparedTraces' `test.jsonl` to the SeedDataset unchanged,
so without one the test split is empty.

Both jobs take overrides on the same PreparedTraces, which re-stages nothing:

```python
config = config_of("prepared-traces", prepared_traces_id)
config.setdefault("trace_processing", {})["relevance_filtering"] = True

retry_id = post(
    "/seed-datasets/from-prepared-traces",
    {"from": prepared_traces_id, "config": config},
)["id"]
poll("seed-datasets", retry_id, 60 * 45)
```

A PreparedTraces holds the trace file it was staged with, so a different set of traces means
staging a new one. Config changes over the same traces are overrides.

### The SeedDataset

A job-input directory (`../data-preparation/overview.md`), staged file by file. Omit
`unstructured_data` for tasks that do not use it. `train_data` and `test_data` are always
staged, but either file can be empty (`../data-preparation/overview.md` § Empty splits).

```python
body = stage(
    "/staging-seed-datasets-s3-urls",
    {
        "train_data": "train.jsonl",
        "test_data": "test.jsonl",
        "config": "config.yaml",
        "job_description": "job_description.json",
    },
)
seed_dataset_id = post("/seed-datasets", body)["id"]
```

`POST /seed-datasets` validates the bundle and returns 400 with the validation error when
it fails. That is this backend's dryrun, and `raise_with_body` is what makes the message
visible instead of a bare `HTTPError`.

The route is metered (`seed_datasets_post`), but only a successful create spends a credit. The
balance is checked first, and the call is recorded only after validation passes. So validating
a broken bundle repeatedly is free, and a 402 here means the balance was already zero before
the bundle was ever read.

### The TrainingDataset from staged files

Normally a TrainingDataset is *produced* by synthgen. It can also be staged directly, which is
how `../../stages/model-training.md` § Dealing with OOM trains without the longest rows. Same
three-step exchange as the SeedDataset, same `STAGING_FIELDS`:

```python
body = stage(
    "/staging-training-datasets-s3-urls",
    {
        "train_data": "train-truncated.jsonl",
        "test_data": "test-truncated.jsonl",
        "config": "config.yaml",
        "job_description": "job_description.json",
    },
)
dataset_id = post("/training-datasets", body)["id"]
```

The create is synchronous and metered on `training_datasets_post`. Getting the rows means
`GET /training-datasets/<id>/download`, metered on `training_datasets_download_get`.

Validation runs the same rules as `POST /seed-datasets`
(`../data-preparation/overview.md` § Validation rules) and returns 400 with the failure.

### The SLM from staged files

An SLM is normally *produced* by training. An existing one, such as a `model.tar` and
`config.yaml` downloaded from another SLM, is registered by staging the two files. This trains
nothing:

```python
body = stage(
    "/staging-slms-s3-urls",
    {"model": "model.tar", "config": "config.yaml"},
)
slm_id = post("/slms", body)["id"]
poll("slms", slm_id, 60 * 30)
```

The tarball must hold the LoRA adapter under `model-adapter/`, as `GET /slms/<id>/download`
produces it (`../deployment.md` § Artifacts); the config travels beside it, not inside. The
platform expands the tarball into the layout a trained SLM has, which is the job `poll()`
waits on. Files that do not form a valid SLM answer 400. The route is metered on `slms_post`,
which starts at zero, so it needs a grant before the first upload (`../platform.md` § Credits).

### Overrides: how a job is parameterised

Five job creates each accept optional `config` and `job_description` objects inline, alongside
`from`: `POST /prepared-traces/with-expanded-test-set`, `POST /seed-datasets/from-prepared-traces`,
`POST /teacher-evaluations/from-seed-datasets`, `POST /training-datasets/from-seed-datasets`
and `POST /slms/from-training-datasets`, and the two `-smoke` routes as well. The two are
independent, and there is no staging step for either.

Each replaces the parent's file whole, and anything you leave out reverts to a library default
(`../platform.md` § Overrides). So an override is read-edit-resend, never hand-built:

```
config_of(collection, id)  →  edit one field  →  POST it whole
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
parent = config_of("seed-datasets", seed_dataset_id)
config = json.loads(json.dumps(parent))          # a copy to edit
config.setdefault("synthgen", {})["generation_target"] = 512

dropped = {
    f"{section}.{key}"
    for section, values in parent.items() if isinstance(values, dict)
    for key in values if key not in config.get(section, {})
}
assert not dropped, f"these would revert to defaults: {dropped}"
```

Errors: 400 for a config that fails validation or a bundle that does not parse (carrying the
validation errors), 404 for a missing parent, 409 for a parent that is not ready, 402 for
insufficient credits.

### Teacher evaluation

```python
teacher_evaluation_id = post(
    "/teacher-evaluations/from-seed-datasets", {"from": seed_dataset_id}
)["id"]
poll("teacher-evaluations", teacher_evaluation_id, 60 * 30)
```

A second iteration with sharper judge instructions is the same SeedDataset with a
`job_description` override:

```python
job_description = job_description_of("seed-datasets", seed_dataset_id)
job_description["llm_as_a_judge_instructions"] = "<sharper instructions>"

retry_id = post(
    "/teacher-evaluations/from-seed-datasets",
    {"from": seed_dataset_id, "job_description": job_description},
)["id"]
poll("teacher-evaluations", retry_id, 60 * 30)
```

### Synthetic data generation

A smoke is the same submission on the `-smoke` route, which sets `synthgen.generation_target`
to 128 whatever the config says (`../platform.md` § Smoke runs). A config override travels
with it as usual; editing the config read back from the parent is what keeps every other
setting intact:

```python
config = config_of("seed-datasets", seed_dataset_id)
# setdefault because an optional section the parent never set is simply
# absent; it was taking defaults already, so creating it changes nothing.
config.setdefault("synthgen", {})["validation_similarity_threshold"] = 0.9

smoke_id = post(
    "/training-datasets/from-seed-datasets-smoke",
    {"from": seed_dataset_id, "config": config},
)["id"]
poll("training-datasets", smoke_id, 60 * 90)
```

The full run is the last passing smoke's body posted to `/training-datasets/from-seed-datasets`
instead, with the intended `generation_target`.

### Model training

Training reads its baseline config from the TrainingDataset, which inherited it from the
SeedDataset synthgen ran over. Setting sensible `tuning` values before synthgen means the
common case needs no override:

```python
slm_id = post("/slms/from-training-datasets", {"from": dataset_id})["id"]
poll("slms", slm_id, 60 * 90)
```

A memory check before the full run is the same submission on
`/slms/from-training-datasets-smoke`: one epoch on the 128 longest train rows and the 32 longest
test rows (`../platform.md` § Smoke runs). Send it the config the full run will use, since the
student and the batch size are what it checks:

```python
config = config_of("training-datasets", dataset_id)
config["base"]["student_model_name"] = "<student-a>"

smoke_id = post(
    "/slms/from-training-datasets-smoke", {"from": dataset_id, "config": config}
)["id"]
poll("slms", smoke_id, 60 * 90)
```

A sweep is N submissions against the same dataset, one per student. Read the dataset's config
once, then vary one field per submission:

```python
base_config = config_of("training-datasets", dataset_id)
students = ["<student-a>", "<student-b>"]

slm_ids = {}
for student in students:
    config = json.loads(json.dumps(base_config))  # a copy per submission
    config["base"]["student_model_name"] = student
    slm_ids[student] = post(
        "/slms/from-training-datasets",
        {"from": dataset_id, "config": config},
    )["id"]

for student, slm_id in slm_ids.items():
    poll("slms", slm_id, 60 * 90)
```

The deep copy matters: mutating one dict across iterations would send every student the last
one's value. `per_device_train_batch_size`, `memory_optimized_training` and `use_qlora` vary
the same way (read, edit, resend), which is how OOM is handled without touching the data.
Submissions run concurrently, so poll them after they are all in.

## Monitor

Poll `GET /<collection>/<id>/status` every 20 seconds. `poll()` in the preamble does this. The
status values and how to run a poller: `../platform.md` § Job status.

```python
poll("training-datasets", dataset_id, 60 * 90)
```

Deployments report `deployment_status` rather than `status`, so they need
`poll(..., status_field="deployment_status")`.

On failure, `GET /<collection>/<id>/logs` returns the job log, and `poll()` raises with its
tail attached.

`GET /<collection>` lists every entity in a collection, newest first, and recovers a lost id.
Each item has `id`, `created_at`, the parent's id under its own key (`seed_dataset_id`,
`training_dataset_id`, `slm_id`, `prepared_traces_id`, `parent_prepared_traces_id`) and, where
an entity can come from more than one kind of parent, a `source`. The list carries no status;
`GET /<collection>/<id>` returns the same fields with `status` added (`deployment_status` and
`endpoint_status` for a deployment).

```python
slms = get("/slms")
from_dataset = [slm["id"] for slm in slms if slm.get("training_dataset_id") == dataset_id]
```

## Output layout

Nothing is fetched by object-storage path. Outputs arrive either as JSON in the response or as
presigned URLs. A `/download` route returns one URL per file, with `null` for a file the entity
does not have. Which outputs each stage produces, and the conditions on them:
`../platform.md` § What each stage produces.

| Entity | `/metrics` | `/download` | `/download-metadata` | Other |
|---|---|---|---|---|
| PreparedTraces | `base_model_performance`, `base_model_predictions_download_url` | `traces_url`, `config_url`, `job_description_url`, `test_data_url` | free | none |
| SeedDataset | none | `train_data_url`, `test_data_url`, `unstructured_data_url`, `config_url`, `job_description_url` | free | none |
| TeacherEvaluation | `teacher_performance`, `predictions_download_url` | none | free | none |
| TrainingDataset | `train_data_size_bytes`, `test_data_size_bytes` | credit gated | free | `/sample` |
| SLM | `base_model_performance`, `tuned_model_performance`, `predictions_download_url` | `model_url`, `config_url` | free; also `model_client_url` | none |
| Deployment | none | none | none | none |

A PreparedTraces has metrics only when the test set from traces job built it with
`traces_to_test_set.evaluate_original_model: true` (`../platform.md` § What each stage
produces). An uploaded one reads `null` in both fields.

A field that is not ready yet reads `null`. `config_of()` turns that null into an error naming
the entity and its status, because the raw failure (`Invalid URL 'None'`) names neither.

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

# The production model's score on the test set built from traces
traces_metrics = get(f"/prepared-traces/{updated_traces_id}/metrics")
print(traces_metrics["base_model_performance"])
original_predictions = requests.get(
    traces_metrics["base_model_predictions_download_url"]
).text
```

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

### Reading a TrainingDataset

`/sample` is free and returns `{"rows": [...]}`. `/download` returns every file and is metered
on `training_datasets_download_get`. What the sample contains and what it leaves out:
`../platform.md` § What each stage produces.

```python
rows = get(f"/training-datasets/{dataset_id}/sample")["rows"]
size = get(f"/training-datasets/{dataset_id}/metrics")["train_data_size_bytes"]

# A short dataset arrives whole, so the sample IS the count. A longer one
# is truncated, and /metrics reports bytes rather than rows, so the total
# has to be estimated from the mean row size.
if len(rows) < 128:
    print(f"{len(rows)} rows")
else:
    mean_row_bytes = sum(len(json.dumps(row)) + 1 for row in rows) / len(rows)
    print(f"~{round(size / mean_row_bytes)} rows (estimated)")
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

## Reuse synthetic data for training-only runs

Retrain the same dataset by id under a config override. Nothing is downloaded, copied or
reassembled:

```python
config = config_of("training-datasets", dataset_id)
config.setdefault("tuning", {})["num_train_epochs"] = 6

retrained = post(
    "/slms/from-training-datasets", {"from": dataset_id, "config": config}
)["id"]
```

This is the same call as the sweep in § Submitting jobs, with one field changed instead of
the student.

## Deploy (hosted)

```python
deployment_id = post("/deployments/from-slms", {"from": slm_id})["id"]
poll("deployments", deployment_id, 60 * 40, status_field="deployment_status")
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

## Inference endpoints (collecting traces)

An inference endpoint is an OpenAI-compatible gateway in front of the model the user already runs
in production. Their application calls the endpoint instead of the provider, the fallback model
answers as before, and the platform keeps a copy of every call. Those copies are the traces that
`stages/trace-processing.md` consumes, so this is the route for a user who wants a distilled
model but has no trace file to start from.

This is not a Deployment. It is permanent, has no idle timeout, cannot be edited or deleted, and
serves no trained model of its own until one is set as its primary (§ Serve the student behind an
endpoint).

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

### The call the user has to make

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
traces accumulate at the rate of their traffic, and trace processing wants hundreds, so the next
stage is days or weeks away rather than minutes. Say that plainly when proposing this route.

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

**What comes back is not a trace processing input.** It is the platform's record of each call:
identifiers, timings and metadata around the request and the response. Trace processing wants one
`{"messages": [...]}` object per line. Inspect a record before writing any conversion, convert per
`../data-preparation/traces.md` § From an inference endpoint, then stage the result as
`traces_jsonl` like any other trace file.

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
record of the file before writing a conversion, as with `/traces`: it is not a trace
processing input until converted.

### Traces from an endpoint

`POST /prepared-traces` also takes the endpoint's records directly, with nothing downloaded or
converted. Stage the config and job description (and a test file, if there is one), then name
the endpoint in place of `traces_jsonl`:

```python
body = stage(
    "/staging-prepared-traces-s3-urls",
    {"config": "config.yaml", "job_description_json": "job_description.json"},
    PREPARED_TRACES_FIELDS,
)
body["inference_endpoint_name"] = name
prepared_traces_id = post("/prepared-traces", body)["id"]
```

Send `traces_jsonl` or `inference_endpoint_name`, not both. Set
`trace_processing.observation_format: langfuse` in the config, since the records are not
`{"messages": [...]}` lines. Which records go in, how fresh they are, and when to take this
route over the download: `../inference-endpoints.md` § From records to a traces object. The
result is an uploaded PreparedTraces like any other, so the test set from traces job runs on it
next (§ Test set from traces and trace processing).

### Serve the student behind an endpoint

The same route puts a trained model in front of the traffic. Deploy the student (§ Deploy
(hosted)), wait for `JOB_SUCCESS`, then create a new endpoint with the deployment id as
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
