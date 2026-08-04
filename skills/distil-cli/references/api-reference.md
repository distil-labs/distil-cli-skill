# API Reference

Use the Distil Labs REST API to integrate the platform into scripts, pipelines, and applications. The API mirrors the CLI workflow: create a model, upload data, evaluate, train, and deploy.

**Base URL:** `https://api.distillabs.ai`

---

## Authentication Setup

All API requests require a Bearer token in the `Authorization` header. Tokens are obtained via AWS Cognito using your Distil Labs username and password.

### 1. Create an account

```bash
# Via CLI
distil register

# Or via web app
# Visit https://app.distillabs.ai/sign-up
```

### 2. Get a Bearer token

```python
import os
import json
import requests

DISTIL_USERNAME = os.environ.get("DISTIL_USERNAME", "")
DISTIL_PASSWORD = os.environ.get("DISTIL_PASSWORD", "")

if not DISTIL_USERNAME or not DISTIL_PASSWORD:
    raise ValueError("DISTIL_USERNAME and DISTIL_PASSWORD must be set")


def distil_bearer_token() -> str:
    """Get an authentication token from the Distil Labs API."""
    response = requests.post(
        "https://cognito-idp.eu-central-1.amazonaws.com",
        headers={
            "X-Amz-Target": "AWSCognitoIdentityProviderService.InitiateAuth",
            "Content-Type": "application/x-amz-json-1.1",
        },
        data=json.dumps({
            "AuthParameters": {"USERNAME": DISTIL_USERNAME, "PASSWORD": DISTIL_PASSWORD},
            "AuthFlow": "USER_PASSWORD_AUTH",
            "ClientId": "4569nvlkn8dm0iedo54nbta6fd",
        })
    )
    response.raise_for_status()
    return response.json()["AuthenticationResult"]["AccessToken"]
```

### 3. Use the token

```python
auth_header = {"Authorization": f"Bearer {distil_bearer_token()}"}
```

Tokens expire after **1 hour**. Your code should handle re-authentication when needed.

### Troubleshooting

- **401 Unauthorized**: Check that your username and password are correct.
- **403 Forbidden**: Token has expired — generate a new one.
- **Connection Errors**: Ensure network connectivity to `api.distillabs.ai` and `cognito-idp.eu-central-1.amazonaws.com`.

For account issues, contact [contact@distillabs.ai](mailto:contact@distillabs.ai).

---

## Endpoint Reference

### Create a model

```python
response = requests.post(
    "https://api.distillabs.ai/models",
    data=json.dumps({"name": "my-model-name"}),
    headers={"Content-Type": "application/json", **auth_header},
)
model_id = response.json()["id"]
```

### List models

```python
response = requests.get(
    "https://api.distillabs.ai/models",
    headers=auth_header,
)
```

### Upload dataset

```python
data = {
    "job_description": {"type": "json", "content": open("data/job_description.json").read()},
    "train_data": {"type": "jsonl", "content": open("data/train.jsonl").read()},
    "test_data": {"type": "jsonl", "content": open("data/test.jsonl").read()},
    "unstructured_data": {"type": "jsonl", "content": open("data/unstructured.jsonl").read()},
    "config": {"type": "yaml", "content": open("data/config.yaml").read()},
}
response = requests.post(
    f"https://api.distillabs.ai/models/{model_id}/uploads",
    data=json.dumps(data),
    headers={"Content-Type": "application/json", **auth_header},
)
upload_id = response.json()["id"]
```

### Upload traces

Three steps: stage the files in S3, register them as prepared traces, then start processing.

```python
# Step 1: Get presigned S3 URLs and upload files
response = requests.get(
    "https://api.distillabs.ai/staging-prepared-traces-s3-urls",
    headers=auth_header,
)
urls = response.json()

requests.put(urls["traces_jsonl"], data=open("traces/traces.jsonl").read())
requests.put(urls["job_description_json"], data=open("traces/job_description.json").read())
requests.put(urls["config"], data=open("traces/config.yaml").read())

# Step 2: Register the staged files as prepared traces
response = requests.post(
    "https://api.distillabs.ai/prepared-traces",
    data=json.dumps({
        "traces_jsonl": urls["traces_jsonl"],
        "job_description_json": urls["job_description_json"],
        "config": urls["config"],
    }),
    headers={"Content-Type": "application/json", **auth_header},
)
prepared_traces_id = response.json()["id"]

# Step 3: Kick off trace processing
# `config` is merged over the prepared traces' own config on top-level keys;
# `job_description` replaces theirs outright. Both optional.
response = requests.post(
    "https://api.distillabs.ai/uploads/from-prepared-traces",
    data=json.dumps({"from": prepared_traces_id}),
    headers={"Content-Type": "application/json", **auth_header},
)
upload_id = response.json()["id"]
```

Note the staging response key is `config`, not `config_yaml`, and neither endpoint takes a model ID.

### Check upload status

```python
response = requests.get(
    f"https://api.distillabs.ai/uploads/{upload_id}/status",
    headers=auth_header,
)
```

The response is `{"status": …}` only. Base model metrics and logs live on separate endpoints:

```python
# base_model_performance, base_model_predictions_download_url
requests.get(f"https://api.distillabs.ai/uploads/{upload_id}/metrics", headers=auth_header)

# logs of the processing job
requests.get(f"https://api.distillabs.ai/uploads/{upload_id}/logs", headers=auth_header)

# presigned URLs for train / test / unstructured / config / job description
# 409 while the upload is still processing
requests.get(f"https://api.distillabs.ai/uploads/{upload_id}/download", headers=auth_header)

# list uploads / prepared traces, reverse chronological
requests.get("https://api.distillabs.ai/uploads", params={"start": 0, "count": 100}, headers=auth_header)
requests.get("https://api.distillabs.ai/prepared-traces", params={"start": 0, "count": 100}, headers=auth_header)
```

### Run teacher evaluation

```python
import time

response = requests.post(
    "https://api.distillabs.ai/teacher-evaluations/from-uploads",
    data=json.dumps({"from": upload_id}),
    headers={"Content-Type": "application/json", **auth_header},
)
eval_job_id = response.json()["id"]

# Poll for completion
running = True
while running:
    response = requests.get(
        f"https://api.distillabs.ai/teacher-evaluations/{eval_job_id}/status",
        headers=auth_header,
    )
    status = response.json()["status"]
    if status in ("JOB_SUCCESS", "JOB_FAILURE", "JOB_STOPPED"):
        running = False
    print(f"Evaluation status: {status}")
    time.sleep(60)

# Scores and the predictions URL live on /metrics, not /status
metrics = requests.get(
    f"https://api.distillabs.ai/teacher-evaluations/{eval_job_id}/metrics",
    headers=auth_header,
).json()
print(metrics["teacher_performance"])
```

```python
# list teacher evaluations, reverse chronological
requests.get(
    "https://api.distillabs.ai/teacher-evaluations",
    params={"start": 0, "count": 100},
    headers=auth_header,
)

# one teacher evaluation, with its status and the upload it was built from
requests.get(
    f"https://api.distillabs.ai/teacher-evaluations/{eval_job_id}", headers=auth_header
)

# logs of the evaluation job
requests.get(
    f"https://api.distillabs.ai/teacher-evaluations/{eval_job_id}/logs",
    headers=auth_header,
)
```

### Generate a training dataset

Optional. Runs synthetic data generation over an upload as its own step, so the
generated rows can be inspected before training. Training still runs from the
upload; it does not accept a training dataset ID.

```python
# start generation (costs 2 credits)
response = requests.post(
    "https://api.distillabs.ai/training-datasets/from-uploads",
    data=json.dumps({"from": upload_id}),
    headers={"Content-Type": "application/json", **auth_header},
)
training_dataset_id = response.json()["id"]

# poll for completion
running = True
while running:
    status = requests.get(
        f"https://api.distillabs.ai/training-datasets/{training_dataset_id}/status",
        headers=auth_header,
    ).json()["status"]
    if status not in ("JOB_NOT_STARTED", "JOB_PENDING", "JOB_RUNNING"):
        running = False
    else:
        time.sleep(60)

# preview up to 20 rows -- free, deterministic, seeded on the dataset id
rows = requests.get(
    f"https://api.distillabs.ai/training-datasets/{training_dataset_id}/sample",
    headers=auth_header,
).json()["rows"]
```

```python
# list training datasets, reverse chronological
requests.get(
    "https://api.distillabs.ai/training-datasets",
    params={"start": 0, "count": 100},
    headers=auth_header,
)

# one dataset, with its status and the upload it was generated from
requests.get(
    f"https://api.distillabs.ai/training-datasets/{training_dataset_id}",
    headers=auth_header,
)

# stored file sizes -- not job output, generation writes no metrics artifact
requests.get(
    f"https://api.distillabs.ai/training-datasets/{training_dataset_id}/metrics",
    headers=auth_header,
)

# presigned download URLs -- credit-gated, 0 credits granted by default
requests.get(
    f"https://api.distillabs.ai/training-datasets/{training_dataset_id}/download",
    headers=auth_header,
)
```

A dataset created directly from staged files is synchronous and has no job behind
it, so its `status` comes back as `JOB_SUCCESS` straight away. The field is
marked optional in the OpenAPI schema only because it carries a default there —
the API always sends it.

### Run training

```python
response = requests.post(
    f"https://api.distillabs.ai/models/{model_id}/training",
    data=json.dumps({"upload_id": upload_id}),
    headers={"Content-Type": "application/json", **auth_header},
)
training_job_id = response.json()["id"]

# Poll for completion
running = True
while running:
    response = requests.get(
        f"https://api.distillabs.ai/trainings/{training_job_id}/status",
        headers=auth_header,
    )
    status = response.json()["status"]
    if status in ("JOB_SUCCESS", "JOB_FAILURE", "JOB_STOPPED"):
        running = False
    print(f"Training status: {status}")
    time.sleep(60)
```

Job status values: `JOB_NOT_STARTED`, `JOB_PENDING`, `JOB_RUNNING`, `JOB_SUCCESS`, `JOB_FAILURE`, `JOB_STOPPED`. Terminal values are `JOB_SUCCESS`, `JOB_FAILURE`, and `JOB_STOPPED` — anything else means the job is still in progress.

### Get evaluation results

```python
response = requests.get(
    f"https://api.distillabs.ai/trainings/{training_job_id}/evaluation-results",
    headers=auth_header,
)
```

### Download predictions (test set)

```python
response = requests.get(
    f"https://api.distillabs.ai/trainings/{training_job_id}/evaluation-results",
    headers=auth_header,
)
predictions_url = response.json()["finetuned_student_evaluation_predictions_download_url"]

# Download and read
import pandas as pd
pd.read_json("finetuned_student_evaluation_predictions.jsonl", lines=True)
```

### Download model

```python
response = requests.get(
    f"https://api.distillabs.ai/trainings/{training_job_id}/model",
    headers=auth_header,
)
download_url = response.json()
```

### Deploy remotely

```python
# Activate
response = requests.post(
    f"https://api.distillabs.ai/trainings/{training_job_id}/deployment",
    headers={"Content-Type": "application/json", **auth_header},
    json={},
)
deployment = response.json()
# Response includes: id, deployment_status, url, secrets.api_key, client_script
```

Deployment status values: `building`, `active`, `inactive`, `credits_exhausted`.

### Get deployment info

```python
response = requests.get(
    f"https://api.distillabs.ai/trainings/{training_job_id}/deployment",
    headers=auth_header,
)
```

### Deactivate deployment

```python
response = requests.delete(
    f"https://api.distillabs.ai/trainings/{training_job_id}/deployment",
    headers=auth_header,
)
# Returns 204 No Content on success
```

---

## Notes

- **Inference deployments** are for testing, not production. Contact [contact@distillabs.ai](mailto:contact@distillabs.ai) for production deployment.
- **Credits**: All users get $30 of free starting credits for remote inference. Reach out when you need more.
- **Prompt format matters**: When querying a deployed SLM, use the exact system prompt and message format from the `client_script` in the deployment response. SLMs are specialized and expect the same format seen during training.
