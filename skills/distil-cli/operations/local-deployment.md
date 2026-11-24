# Operation: Local Deployment

Serves a trained SLM with vLLM on the user's own GPU and checks it on a handful of test-set rows.
vLLM loads the student's base model from HuggingFace and applies the trained LoRA adapter on top
of it. Nothing records the calls, so local traffic does not become traces.

| | |
|---|---|
| **Called by** | the deploy phase, when the user chooses to serve on their own GPU |
| **Price** | free: downloading the model and serving it locally are not billed |

## Settings that control the output

| Setting | Where | What it changes |
|---|---|---|
| `<huggingface-base-model>` | `vllm serve` | The base model, from `base.student_model_name` in the model's `config.yaml` (`../references/deployment.md` § Serving locally, the table) |
| `--lora-modules model=./model-adapter` | `vllm serve` | Serves the adapter under the name `model`, the name `model_client.py` sends; the untrained base answers under `base` |
| `--max-lora-rank` | `vllm serve` | At least `tuning.lora_r` from the model's `config.yaml` (default 64) |
| the family flags | `vllm serve` | The tool-call and reasoning parsers of the student's family, the same the hosted deployment uses |
| the system prompt, sampling, thinking | each request | Sent by `model_client.py` as in training; a hand-built request does not match them |

## Working directory

```
local-deployment/
└── <slm-id>/
    ├── artifacts/   # model.tar extracted, and config.yaml
    └── run.md       # the serve command, the port, and the smoke-test results
```

## Step 1: Prepare the input

Download the model tarball and its `config.yaml`, then extract the tarball
(`../references/execution/cli.md` § Fetch model artifacts):

```bash
distil slm download --destination local-deployment/<slm-id>/artifacts <slm-id>
tar -xf local-deployment/<slm-id>/artifacts/model.tar -C local-deployment/<slm-id>/artifacts
```

The command checks free disk space before it writes. Both `model-adapter/` and `config.yaml` are
needed: the config is not inside the tarball, and it names the base model and `lora_r`
(`../references/deployment.md` § Artifacts).

The machine needs a GPU that fits the student, vLLM installed, and access to the base model on
HuggingFace. The Llama and Gemma base models are gated: accept the license on the model page and
export `HF_TOKEN` before serving.

## Step 2: Price it

Free. Nothing to add to the run log's spending.

## Step 3: Submit

Start vLLM from the artifacts directory per `../references/deployment.md` § Serving locally:

```bash
vllm serve <huggingface-base-model> \
  --served-model-name base \
  --enable-lora \
  --lora-modules model=./model-adapter \
  --max-lora-rank <tuning.lora_r> \
  --api-key EMPTY \
  <family flags>                          # port 8000
```

Record the command and the port in `run.md`. For a reasoning student, the reasoning parser for
the family is required, or the thinking block stays at the start of `content`
(`../references/reasoning-models.md` § Deployment).

## Step 4: Wait and pull

The server is ready when it accepts requests on the port. Query it only through the model's own
client, never with hand-built requests (`../references/deployment.md` § Why model_client.py
instead of raw requests):

```bash
python local-deployment/<slm-id>/artifacts/model_client.py \
  --conversation '[{"role": "system", "content": "<system prompt>"}, {"role": "user", "content": "…"}]'
```

The client's default base URL is `http://127.0.0.1:8000/v1`; it sends the conversation as given
and raises `ValueError` without a system prompt.

## Step 5: Read the output

Send a handful of test-set rows, each with the system prompt the model was trained with, and
compare the answers with the references. Record the results in `run.md`.

When quality is worse than the evaluation promised, check in order: the prompt format (the system
prompt is the trained one), that the request names `model` and not `base`, and for a model trained
without thinking, that no reasoning text appears in the answer
(`../references/deployment.md` § Why model_client.py instead of raw requests). Reasoning text in
the answer is expected from a reasoning student.
