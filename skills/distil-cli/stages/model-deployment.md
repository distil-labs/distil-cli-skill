# Stage: Model Deployment

Serves the trained model and smoke-tests it on test-set rows. Three routes, and Step 1 chooses
between them: hosted on the platform, local vLLM on downloaded weights, or hosted and then put
behind an inference endpoint so the production application can call it. No smoke/full split on
any of them. One directory per trained model.

## Working Directory

```
model-deployment/
└── <training-id>/
    ├── artifacts/       # downloaded model outputs (local route only)
    ├── run.md           # route, endpoint or serve command, port, deployment id,
    │                    # and the unique endpoint name on the endpoint route
    └── analysis.md      # smoke-test results
```

## Step 1: Choose the Route

Present both to the user with their costs, and record the choice in `run.md`:

| | Hosted | Local | Behind the inference endpoint |
|---|---|---|---|
| What it is | The platform serves the model behind a URL and API key | You download `model.tar` and run vLLM yourself | Hosted, plus a new inference endpoint with the deployment as primary and the production model as fallback |
| Credits | `deployments_from_slms_post`, of which a new account has 2 | none | the same, plus one `inference_endpoints_post` |
| Other cost | **Bills until its idle timeout.** Delete it when finished | a GPU, and the download | the deployment bills as on the hosted route; once it stops, the fallback answers |
| Use it when | there is no local GPU, or the endpoint is the deliverable | iterating locally, or no deployment credits | the model came from production traffic and should now take that traffic (`../workflows/endpoint-to-model.md` Step 8) |

## Step 2a: Deploy (hosted)

Create the deployment and wait for it (the execution backend § Deploy (hosted)). Deployments
report `deployment_status` rather than `status`, and the job does not return until vLLM
answers, so success means serving, not merely scheduled. Record the deployment id and the
endpoint URL in `run.md`.

**Delete the deployment when the smoke test is done.** It bills until its idle timeout, and
the delete is one call. Note it in `run.md` when done, so nothing is left running.

## Step 2b: Serve (local)

Fetch the artifacts (the execution backend § Fetch model artifacts, with the contents listed
in `../references/deployment.md` § Artifacts), then serve per `../references/deployment.md`
§ Serving locally: vLLM on the merged weights. Record the command and port in `run.md`.

## Step 2c: Serve behind the inference endpoint

Do Step 2a first and run the Step 3 smoke test against the deployment directly. Only a
deployment that answers correctly on its own goes behind an endpoint: behind it, a request the
primary cannot serve is answered by the fallback.

Then create the endpoint with the deployment as primary and link a key: the execution backend
§ Serve the student behind an endpoint. Record the unique endpoint name in `run.md`. Send one
request through it and confirm the answer is the student's; each record's `metadata.source`
names which model answered. Then the user moves their application's `model` string to the new
name. The system prompt they send stays the one the traces were collected with.

Tell the user two things before they move traffic: the deployment behind the primary stops on
its own, and from then on every call goes to the fallback; and the new endpoint records its
traffic, so the next iteration's traces come from it.

## Step 3: Smoke Test

Query via `model_client.py` with a handful of test-set rows and compare against expected
answers. Record the results in `analysis.md`.

**Never hand-roll requests, on either route.** The prompt format must match training (the
system prompt, `temperature: 0` and thinking disabled), and `model_client.py` is what encodes
it. The same client serves both routes: pass `base_url` to point it at a hosted deployment,
omit it for a local server. Fetch it from the SLM's `download-metadata` (free) or take it from
the extracted tarball. On the endpoint route, pass the endpoint's base URL and key and the
unique endpoint name as `--model`.

A hand-built request does not fail loudly. It returns `200` with reasoning text in the answer
and a much worse score than the eval metrics promised. So if quality is worse than expected,
suspect prompt-format mismatch first, then check you are serving the merged `model/` directory
and not the base model.
