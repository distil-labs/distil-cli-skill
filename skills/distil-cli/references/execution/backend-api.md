# Execution Backend: distil labs API

How stages run on the platform. Stage files link here by operation name. Every stage is an
entity created through the REST API, and every entity is created either by staging files or
by running a job over the entity before it. The only prerequisite is a distil labs account.
There is no API route that creates one: sign up with `distil signup` or at
`https://app.distillabs.ai/sign-up`, then use those credentials below. If the sign-up page asks
for an email confirmation, confirm it before the first request. Until it is confirmed, every
request fails with `Email confirmation required`.

`cli.md` is the default backend and this one is the alternative. Take it when the user
prefers it, when the work is already scripted in Python, or when the CLI cannot be installed.
`README.md` § Choose the backend decides between them, and the choice goes in `run.md`.

## Prerequisites

```bash
pip install requests pyyaml
export DL_USERNAME="you@example.com"
export DL_PASSWORD="…"
```

The API is `https://api.distillabs.ai` with Cognito client id `4569nvlkn8dm0iedo54nbta6fd`
in `eu-central-1`. Override both for development:

```bash
export DL_PLATFORM_URL=https://api-dev.distillabs.ai
export DL_COGNITO_CLIENT_ID=3b7cng3qhts9cgglmbh854vplc
```

Access tokens last one hour, which is shorter than a synthgen or training run, so the
preamble below re-authenticates per request rather than holding a token. `pyyaml` is needed
because a config is read back as `config.yaml` text and sent as a JSON object.

## Preamble

Every snippet in this file assumes this block.

```python
import json
import os
import time
from pathlib import Path

import requests
import yaml

PLATFORM_URL = os.getenv("DL_PLATFORM_URL", "https://api.distillabs.ai")
COGNITO_CLIENT_ID = os.getenv(
    "DL_COGNITO_CLIENT_ID", "4569nvlkn8dm0iedo54nbta6fd"
)
COGNITO_URL = "https://cognito-idp.eu-central-1.amazonaws.com"
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
    """Tokens last an hour, which is shorter than a training run, so this is
    called per request rather than cached."""
    response = requests.post(
        COGNITO_URL,
        headers={
            "X-Amz-Target": "AWSCognitoIdentityProviderService.InitiateAuth",
            "Content-Type": "application/x-amz-json-1.1",
        },
        data=json.dumps(
            {
                "AuthParameters": {
                    "USERNAME": os.environ["DL_USERNAME"],
                    "PASSWORD": os.environ["DL_PASSWORD"],
                },
                "AuthFlow": "USER_PASSWORD_AUTH",
                "ClientId": COGNITO_CLIENT_ID,
            }
        ),
    )
    response.raise_for_status()
    token = response.json()["AuthenticationResult"]["AccessToken"]
    return {"Authorization": token}


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
| trace-processing | PreparedTraces → SeedDataset | `POST /prepared-traces`, then `POST /seed-datasets/from-prepared-traces` | `GET /seed-datasets/<id>/{status,logs,metrics,download,download-metadata}` | yes |
| (job input) | SeedDataset | `POST /seed-datasets` | `GET /seed-datasets/<id>/{status,logs,metrics,download,download-metadata}` | staged files only |
| teacher-evaluation | TeacherEvaluation | `POST /teacher-evaluations/from-seed-datasets` | `GET /teacher-evaluations/<id>/{status,logs,metrics,download-metadata}` | yes |
| synthetic-data-generation | TrainingDataset | `POST /training-datasets/from-seed-datasets`, or `POST /training-datasets` from staged files | `GET /training-datasets/<id>/{status,logs,metrics,sample,download,download-metadata}` | yes |
| test-set-expansion | TrainingDataset | as synthgen, from a **new** SeedDataset with train/test inverted | as synthgen | data change, so no |
| model-training | SLM | `POST /slms/from-training-datasets` | `GET /slms/<id>/{status,logs,metrics,download,download-metadata}` | yes |
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
with `.get()`. A submission against an exhausted route fails 402.

## Submitting jobs

Every job is created by posting the id of the entity before it. Nothing is uploaded at this
point, because the parent's files are already on the platform.

| Stage | Collection | Typical timeout |
|---|---|---|
| Trace processing | `seed-datasets` | 45 min |
| Teacher evaluation | `teacher-evaluations` | 30 min |
| Synthetic data generation | `training-datasets` | 90 min |
| Model training | `slms` | 90 min |
| Deployment | `deployments` | 40 min |

### Trace processing

Stage the traces, then process them into a SeedDataset.

```python
body = stage(
    "/staging-prepared-traces-s3-urls",
    {
        "traces_jsonl": "traces.jsonl",
        "config": "config.yaml",
        "job_description_json": "job_description.json",
        # Add "test_jsonl" only for a curated test set: supplying one
        # replaces the generated test split and makes
        # num_traces_as_testing_base inert.
    },
    PREPARED_TRACES_FIELDS,
)
prepared_traces_id = post("/prepared-traces", body)["id"]

seed_dataset_id = post(
    "/seed-datasets/from-prepared-traces", {"from": prepared_traces_id}
)["id"]
poll("seed-datasets", seed_dataset_id, 60 * 45)
```

Iterating is an override on the same PreparedTraces, which re-stages nothing:

```python
config = config_of("prepared-traces", prepared_traces_id)
config["trace_processing"]["relevance_filtering"] = True

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

### The TrainingDataset (training smoke calibration)

Normally a TrainingDataset is *produced* by synthgen. It can also be staged directly, which is
how `../../stages/model-training.md` Step 3 builds the truncated dataset it calibrates memory
settings on. Same three-step exchange as the SeedDataset, same `STAGING_FIELDS`:

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

### Overrides: how a job is parameterised

Four job creates each accept optional `config` and `job_description` objects inline, alongside
`from`: `POST /seed-datasets/from-prepared-traces`,
`POST /teacher-evaluations/from-seed-datasets`, `POST /training-datasets/from-seed-datasets`
and `POST /slms/from-training-datasets`. The two are independent, and there is no staging step
for either.

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
config["synthgen"]["generation_target"] = 512

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

A smoke run is a config override on the same SeedDataset. Editing the config read back from
the parent is what keeps every other setting intact:

```python
config = config_of("seed-datasets", seed_dataset_id)
# setdefault because an optional section the parent never set is simply
# absent; it was taking defaults already, so creating it changes nothing.
config.setdefault("synthgen", {})["generation_target"] = 64
config["synthgen"]["generation_iteration_size"] = 16

dataset_id = post(
    "/training-datasets/from-seed-datasets",
    {"from": seed_dataset_id, "config": config},
)["id"]
poll("training-datasets", dataset_id, 60 * 90)
```

The full run is the same call with the intended `generation_target` and
`generation_iteration_size`, or with no `config` at all if the SeedDataset already carries
them.

### Model training

Training reads its baseline config from the TrainingDataset, which inherited it from the
SeedDataset synthgen ran over. Setting sensible `tuning` values before synthgen means the
common case needs no override:

```python
slm_id = post("/slms/from-training-datasets", {"from": dataset_id})["id"]
poll("slms", slm_id, 60 * 90)
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

## Output layout

Nothing is fetched by object-storage path. Outputs arrive either as JSON in the response or as
presigned URLs. A `/download` route returns one URL per file, with `null` for a file the entity
does not have. Which outputs each stage produces, and the conditions on them:
`../platform.md` § What each stage produces.

| Entity | `/metrics` | `/download` | `/download-metadata` | Other |
|---|---|---|---|---|
| PreparedTraces | none | `traces_url`, `config_url`, `job_description_url`, `test_data_url` | free | none |
| SeedDataset | `base_model_performance`, `base_model_predictions_download_url` | `train_data_url`, `test_data_url`, `unstructured_data_url`, `config_url`, `job_description_url` | free | none |
| TeacherEvaluation | `teacher_performance`, `predictions_download_url` | none | free | none |
| TrainingDataset | `train_data_size_bytes`, `test_data_size_bytes` | credit gated | free | `/sample` |
| SLM | `base_model_performance`, `tuned_model_performance`, `predictions_download_url` | `model_url`, `config_url` | free; also `model_client_url` | none |
| Deployment | none | none | none | `/endpoint` |

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

endpoint = get(f"/deployments/{deployment_id}/endpoint")
```

`endpoint` carries the `url` and the `api_key`. Query the deployment through the model's own
client, not a hand-built request: `../deployment.md` § Serving hosted has the call and says
why.

```python
requests.delete(f"{PLATFORM_URL}/deployments/{deployment_id}", headers=auth())
```

The job does not return until vLLM answers, so `JOB_SUCCESS` means serving rather than merely
scheduled. After the delete, `deployment_status` stays `JOB_SUCCESS`. `endpoint_status` going
to `stopped` is what confirms the deployment is down. Delete the deployment when finished and
check that field, because a running deployment bills until its idle timeout.
