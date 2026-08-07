# Deployment and Integration

Serve a trained SLM either on Distil Labs hosted infrastructure or on your own machine.

Both paths start from an `<slm-id>`. Get it from `distil slm create-from-training-dataset`, or find it later with `distil slm list --output json | jq -r '.[0].id'`. Confirm the SLM is ready before deploying:

```bash
distil slm status <slm-id> --output json | jq -r '.status'   # want JOB_SUCCESS
```

## Which Path?

| | Hosted deployment | Local serving |
|---|---|---|
| Command | `distil deployment create-from-slm <slm-id>` | `distil slm download <slm-id>` |
| Setup | none | you install and run the inference server |
| Cost | consumes inference credits while running | free |
| Good for | quick testing, sharing an endpoint, integration work | offline use, air-gapped environments, high-volume serving |

Hosted deployments are not intended for production use — contact contact@distillabs.ai when you are ready for production.

## Hosted Deployment

### Create the deployment

```bash
distil deployment create-from-slm <slm-id>
# Output: Deployment started. Deployment ID: <deployment-id>
```

Capture the `<deployment-id>`. Provisioning is asynchronous.

### Wait for it to serve

`distil deployment status` reports two independent things. Poll on `endpoint_status`, not `deployment_status` — a finished deploy whose endpoint is still `stopped` cannot answer a request yet.

```bash
while true; do
    endpoint=$(distil deployment status <deployment-id> --output json | jq -r '.endpoint_status // "none"')
    echo "endpoint: $endpoint"
    if [ "$endpoint" = "running" ]; then break; fi
    sleep 30
done
```

See `references/tasks/polling-jobs.md` for the general polling rules.

### Get the URL and API key

```bash
distil deployment endpoint <deployment-id>
distil deployment endpoint <deployment-id> --output json
```

Both fields are `null` until the deployment is serving, so this is safe to call while you wait:

```bash
url=$(distil deployment endpoint <deployment-id> --output json | jq -r '.url // empty')
key=$(distil deployment endpoint <deployment-id> --output json | jq -r '.api_key // empty')
```

### Shut it down

A running deployment consumes inference credits. Shut it down when you are done testing:

```bash
distil deployment delete <deployment-id>      # alias: distil deployment shutdown
```

The SLM is untouched — only the serving infrastructure goes away. Redeploy the same SLM later with `distil deployment create-from-slm <slm-id>`.

## Local Serving

### Download and extract

```bash
distil slm download <slm-id> --destination ./my-slm
tar -xf ./my-slm/model.tar -C ./my-slm
```

`distil slm download` writes `model.tar` and the `config.yaml` the SLM was trained from. The tarball holds exactly two directories:

| Path | Contents |
|------|----------|
| `model/` | The model weights, in Hugging Face format. |
| `model-adapter/` | The LoRA adapter. |

There is no client script and no README inside — those came with the older model-scoped download and are not part of an SLM artifact.

### Option 1: vLLM (recommended)

vLLM reads Hugging Face format, so it serves the extracted `model/` directory as-is.

```bash
python -m venv serve
source serve/bin/activate
pip install vllm openai

vllm serve ./my-slm/model --api-key EMPTY
```

For tool calling models:

```bash
vllm serve ./my-slm/model --enable-auto-tool-choice --tool-call-parser hermes --api-key EMPTY
```

The server runs in the foreground and exposes an OpenAI-compatible API on port 8000. Run it in a separate window or as a background process.

### Option 2: llama-cpp

llama-cpp needs GGUF, and **the SLM tarball contains no GGUF file** — so this path requires a conversion step first. Use `convert_hf_to_gguf.py` from a [llama.cpp](https://github.com/ggerganov/llama.cpp) checkout:

```bash
python convert_hf_to_gguf.py ./my-slm/model --outfile ./my-slm/model.gguf
llama-server -m ./my-slm/model.gguf --port 8000
```

If you only want a local endpoint and do not specifically need llama-cpp, vLLM is less work.

### Option 3: Ollama

Ollama also wants GGUF. Convert as above, then write a `Modelfile` pointing at the result:

```
FROM ./model.gguf
```

```bash
ollama create my-slm -f Modelfile
ollama run my-slm
```

## Querying Your Model

Both hosted and local deployments expose an OpenAI-compatible API at `/v1`, so any OpenAI-compatible client works.

```python
from openai import OpenAI

# Hosted deployment -- url and api_key from `distil deployment endpoint <deployment-id>`
client = OpenAI(base_url="<url>/v1", api_key="<api-key>")

# Local vLLM
client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

response = client.chat.completions.create(
    model="model",
    messages=[
        {"role": "system", "content": "Your system prompt"},
        {"role": "user", "content": "Your question"},
    ],
)
print(response.choices[0].message.content)
```

With curl against a hosted deployment:

```bash
curl "$url/v1/chat/completions" \
  -H "Authorization: Bearer $key" \
  -H "Content-Type: application/json" \
  -d '{"model": "model", "messages": [{"role": "user", "content": "Your question"}]}'
```

**Important:** Use the same system prompt and message formatting the model saw during training. SLMs are specialized and expect exactly the training-time format — a different system prompt or a reshaped message will degrade quality badly. The `config.yaml` that `distil slm download` writes alongside the tarball records what the model was trained with.

For question answering tasks that take context, wrap it in a `<context>` tag followed by a newline, inside the first user message:

```json
[{"role": "user", "content": "<context>Your context here</context>\nYour question here"}]
```

## Credits

Hosted deployments require credits. All users get $30 of free starting credits. When credits are exhausted you cannot create new deployments, and existing deployments are shut down. Contact contact@distillabs.ai when you need more.

Local serving costs nothing once the SLM is downloaded.
