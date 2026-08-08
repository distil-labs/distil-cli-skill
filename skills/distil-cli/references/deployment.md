# Deployment

Trained-model artifacts, the inference client, and serving. Procedure:
`../stages/model-deployment.md`. Fetching the files: the execution backend (`execution/`).

## Artifacts (training output)

| Artifact | What it is |
|---|---|
| `model/` | Merged finetuned weights (safetensors) |
| `model-adapter/` | LoRA adapter (when `use_lora: true`) |
| `model_client.py` | Generated inference client; the canonical way to query the model |
| `README.md` | Serving instructions (vLLM) |
| `model.tar` | Tarball of the above plus LICENSE / TEACHER_LICENSE / STUDENT_LICENSE |

`model_client.py` also has its own presigned URL, a few kilobytes instead of the several
gigabytes of the tarball, so a client can be fetched without downloading the model. The
execution backend has the call.

Each model carries its own client, generated for it at training time and matching the prompt
shape that model was trained with. Use the client that came with the model you are querying.

## Why model_client.py instead of raw requests

A small model degrades sharply outside its training-time setup, and the client reproduces that
setup exactly:

- The trained system prompt, baked in as `SYSTEM_PROMPT`.
- `temperature=0`, with thinking disabled (`chat_template_kwargs: {"enable_thinking": false}`).
- Tool-calling models call with `tools=TOOLS, tool_choice="required"` and return the first
  tool call.
- QA context inlined into the first user message as `<context>...</context>`, matching training
  and evaluation.

A hand-built chat-completions request matches none of that, and it fails quietly: the endpoint
answers `200`, reasoning text leaks into the answer, and the score falls well below what the
evaluation metrics promised.

So query through the client on both routes below, and smoke-test any deployment with it on a
few test-set rows before integrating.

## The client's interface

```python
DistilLabsLLM(model_name: str, base_url: str, api_key: str = "EMPTY")
```

`base_url` is required. Pass the hosted deployment's URL with `/v1` appended, or a local
server's. `invoke()` takes a list of chat messages and returns the answer:

```python
from model_client import DistilLabsLLM

client = DistilLabsLLM(model_name="model", base_url="http://127.0.0.1:8000/v1")
print(client.invoke([{"role": "user", "content": "..."}]))
```

As a command it takes `--base-url` (default `http://127.0.0.1:8000/v1`), `--api-key`,
`--model`, and `--conversation`. Without `--conversation` it replays a baked-in example from
the test data:

```bash
python model_client.py --conversation '[{"role": "user", "content": "..."}]'
```

## Serving locally

One documented backend, OpenAI-compatible (no Ollama support). Serve the merged weights
directory, then query it with the client:

```bash
vllm serve model --api-key EMPTY          # port 8000

python model_client.py --conversation '[{"role": "user", "content": "..."}]'
```

Both `model/` and the model's `config.yaml` are needed; the config is not inside the tarball.

## Serving hosted

The platform serves the model behind a URL and an API key, which the execution backend's
deployment call returns. Point the same client at it:

```python
client = DistilLabsLLM(
    model_name="model",
    base_url=f"{endpoint['url'].rstrip('/')}/v1",
    api_key=endpoint["api_key"],
)
```

The API key is load-bearing: the tunnel has no authentication of its own and the URL is
otherwise open. A hosted deployment bills until its idle timeout, so delete it when the smoke
test is done.
