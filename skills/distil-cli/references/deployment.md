# Deployment

Trained-model artifacts, the inference client, and serving. Procedures:
`../stages/local-deployment.md` (your own GPU) and `../stages/inference-endpoint.md` (hosted,
behind an inference endpoint). Fetching the files: `execution/cli.md` § Fetch model artifacts.
The vLLM commands in § Serving locally are the one set of commands kept outside
`execution/cli.md`.

## Artifacts

| Artifact | What it is |
|---|---|
| `model-adapter/` | The LoRA adapter (`adapter_config.json` and the adapter weights). This is the trained model. Also holds LICENSE / TEACHER_LICENSE / STUDENT_LICENSE |
| `model_client.py` | Generated inference client; the canonical way to query the model |
| `README.md` | Serving instructions (vLLM) |
| `model.tar` | Tarball of the above |

The adapter is served on the student's base model from HuggingFace, so the tarball holds no
base weights. Which models can be deployed: `inference-endpoints.md` § Lifetime.

Each model has its own `model_client.py`, generated at training time for the prompt shape it
was trained with; it can be fetched without the tarball (`execution/cli.md` § The inference
client on its own).

## Why model_client.py instead of raw requests

A small model degrades sharply outside its training-time setup, and the client reproduces that
setup:

- The trained system prompt, which the client contains as `SYSTEM_PROMPT`.
- `temperature=0`, and the thinking setting of training sent as `chat_template_kwargs`
  (`reasoning-models.md` § Deployment).
- Chat completion models with tools call with `tools=TOOLS, tool_choice="auto"`, so the model
  decides whether to call. `invoke` returns the whole assistant message, content and tool calls
  included.
- The client sends one request per call and does not run the agentic loop: for
  `chat-completion-agentic`, the caller executes the calls, appends the assistant message plus
  one `{"role": "tool", "tool_call_id": ..., "content": ...}` per call, and invokes again.

A hand-built chat completions request matches none of that and does not fail: the server
answers `200`, reasoning text appears in the answer of a model trained without thinking, and
the score falls well below the evaluation metrics. So every new caller and integration queries
through the client, and every deployment is smoke-tested with it on a few test-set rows.

The model must run with the system prompt it was trained with: the job description's
`task_description`. `model_client.py` sends it. An application that calls the endpoint with its
own requests must send that same system prompt, with `temperature=0` and thinking as in
training. When the application's production prompt differs from `task_description`, change the
application's system prompt to it, or call through `model_client.py`.

When quality is worse than expected, check the prompt format first, then that the request
names the adapter (`model`), not the base model (`base`).

## The client's interface

```python
DistilLabsLLM(model_name: str, base_url: str, api_key: str = "EMPTY")
```

`base_url` is required: a local server (`http://127.0.0.1:8000/v1`) or an inference endpoint
(`https://inference.distillabs.ai/v1`). A deployment is never called directly. `invoke()` takes
a list of chat messages and returns the answer:

```python
from model_client import DistilLabsLLM

client = DistilLabsLLM(model_name="model", base_url="http://127.0.0.1:8000/v1")
print(client.invoke([{"role": "user", "content": "..."}]))
```

As a command it takes `--base-url` (default `http://127.0.0.1:8000/v1`), `--api-key`,
`--model`, and `--conversation`. Without `--conversation` it sends an example from the
test data that the client contains:

```bash
python model_client.py --conversation '[{"role": "user", "content": "..."}]'
```

## Serving locally

vLLM is the supported server (no Ollama). It loads the base model from HuggingFace and applies
the adapter on top of it, the same way the hosted deployment serves it. Extract the tarball,
then start the server:

```bash
tar -xf model.tar
vllm serve <huggingface-base-model> \
  --served-model-name base \
  --enable-lora \
  --lora-modules model=./model-adapter \
  --max-lora-rank <tuning.lora_r> \
  --api-key EMPTY \
  <family flags>                          # port 8000

python model_client.py --conversation '[{"role": "user", "content": "..."}]'
```

- `--lora-modules model=./model-adapter` serves the adapter under the name `model`, which is the
  name `model_client.py` sends. The base model answers under `base`, so a request for `model`
  reaches the trained model and a request for `base` the untrained one.
- `--max-lora-rank` must be at least `tuning.lora_r` from the model's `config.yaml` (default 64).
- `<huggingface-base-model>` follows from `base.student_model_name` in the model's
  `config.yaml`:

  | `student_model_name` | HuggingFace base model |
  |---|---|
  | `Llama-3.2-1B-Instruct`, `Llama-3.2-3B-Instruct`, `Llama-3.1-8B-Instruct` | `meta-llama/<name>` |
  | `Qwen3-0.6B`, `Qwen3-1.7B`, `Qwen3-4B-Instruct-2507`, `Qwen3-8B` | `Qwen/<name>` |
  | `Qwen3.5-0.8B`, `Qwen3.5-2B`, `Qwen3.5-4B`, `Qwen3.5-9B`, `Qwen3.6-35B-A3B`, `Qwen3.8-27B` | `Qwen/<name>` |
  | `LFM2-350M`, `LFM2-1.2B`, `LFM2-2.6B`, `LFM2.5-350M`, `LFM2.5-1.2B-Instruct` | `LiquidAI/<name>` |
  | `gemma-3-270m-it`, `gemma-3-1b-it`, `gemma-3-4b-it`, `gemma-4-E2B-it`, `gemma-4-E4B-it`, `functiongemma-270m-it` | `google/<name>` |
  | `SmolLM2-135M-Instruct`, `SmolLM2-1.7B-Instruct` | `HuggingFaceTB/<name>` |

  The Llama and Gemma base models are gated on HuggingFace: accept the license on the model
  page and export `HF_TOKEN` before `vllm serve`.
- `<family flags>` are the tool-call and reasoning parser flags of the student's family. The
  hosted deployment passes the same flags:

  | Student family | Flags |
  |---|---|
  | Qwen3 | `--enable-auto-tool-choice --tool-call-parser hermes --reasoning-parser qwen3` |
  | Qwen3.5, Qwen3.6, Qwen3.8 | `--enable-auto-tool-choice --tool-call-parser qwen3_xml --reasoning-parser qwen3` |
  | Llama 3 | `--enable-auto-tool-choice --tool-call-parser llama3_json` |
  | LFM2, LFM2.5 | `--enable-auto-tool-choice --tool-call-parser lfm2` |
  | Gemma 4 | `--enable-auto-tool-choice --tool-call-parser gemma4 --reasoning-parser gemma4` |
  | FunctionGemma | `--enable-auto-tool-choice --tool-call-parser functiongemma` |
  | Gemma 3, SmolLM2 | none |

Both `model-adapter/` and the model's `config.yaml` are needed: the config is not inside the
tarball, and it names the base model and `lora_r`.

## Serving hosted

The platform serves the model behind an inference endpoint with the deployment as primary
(`inference-endpoints.md`). Point the same client at the endpoint, with the unique endpoint name
as the model and an inference API key linked to it:

```python
client = DistilLabsLLM(
    model_name="<unique-endpoint-name>",
    base_url="https://inference.distillabs.ai/v1",
    api_key="<endpoint-api-key>",
)
```

A request the deployment cannot serve is answered by the fallback, so check `metadata.source`
in the endpoint's records to confirm the student answered. How long the deployment runs and
when to delete it: `inference-endpoints.md` § Lifetime.
