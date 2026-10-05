# Stage: Local Deployment

Serves the trained model with vLLM on the user's own GPU and smoke-tests it on test-set rows.
vLLM loads the student's base model from HuggingFace and applies the trained LoRA adapter on top
of it. One directory per trained model. To serve the model on the platform, in front of the
user's production traffic, use `inference-endpoint.md` instead.

## Working Directory

```
local-deployment/
└── <slm-id>/
    ├── artifacts/       # model.tar extracted, and config.yaml
    ├── run.md           # the serve command and the port
    └── analysis.md      # smoke-test results
```

## Step 1: Prepare the Input

Download the model tarball and its `config.yaml` into `artifacts/`
(`../references/execution/cli.md` § Fetch model artifacts) and extract the tarball. What it holds,
and why the config is needed: `../references/deployment.md` § Artifacts.

## Step 2: Confirm the Setup with the User

Present and confirm: a machine with a GPU that fits the student, vLLM installed on it, and access
to the student's base model on HuggingFace (the Llama and Gemma base models are gated). This
stage spends no credits.

## Step 3: Serve

Start vLLM per `../references/deployment.md` § Serving locally and record the command and the
port in `run.md`.

## Step 4: Smoke Test

Query the server through `model_client.py` with a handful of test-set rows, never with
hand-built requests (`../references/deployment.md` § Why model_client.py instead of raw
requests), and compare against the expected answers. Record the results in `analysis.md`.

If quality is worse than the evaluation promised, follow the checks in
`../references/deployment.md` § Why model_client.py instead of raw requests. For a reasoning
student, see `../references/reasoning-models.md` § Deployment.
