# Deployment and Integration

Download and deploy your trained model locally or on Distil Labs hosted infrastructure.

## Download Trained Model

Download your trained model after training completes:

```bash
distil model download <model-id>
```

### Downloaded Model Structure

The download contains everything needed to run your model:

| File/Directory | Description |
|----------------|-------------|
| `model/` | Model weights. |
| `model-adapters/` | LoRA adapters. |
| `Modelfile` | Model configuration file. |
| `model_client.py` | Inference client script. |
| `README.md` | Usage instructions. |

## Local Deployment

### Option 1: Distil CLI with llama-cpp (Recommended)

Deploy your model locally using the built-in command, which uses llama-cpp as the inference backend:

```bash
distil model deploy local <model-id>
```

This downloads the model and starts a local llama-server on port 8000. The model is available via the OpenAI-compatible API at `http://localhost:8000/v1`.

Customize the port and enable server logs:

```bash
distil model deploy local --port 9000 --logs <model-id>
```

| Option | Description |
|--------|-------------|
| `--port <port>` | Port number for local llama-server (default: 8000). |
| `--logs` | Show llama-server logs during local deployment. |
| `--output json` | Output results in JSON format. |

**Requirement:** Local deployment requires [llama-cpp](https://github.com/ggerganov/llama.cpp) installed on your machine.

### Option 2: vLLM

For high-performance serving, use vLLM:

```bash
pip install vllm openai
vllm serve model --api-key EMPTY
```

For tool calling models:

```bash
vllm serve model --enable-auto-tool-choice --tool-call-parser hermes --api-key EMPTY
```

Query via the OpenAI-compatible API:

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")
response = client.chat.completions.create(
    model="model",
    messages=[{"role": "user", "content": "Your question here"}]
)
print(response.choices[0].message.content)
```

## Querying Your Model

### Using the Invocation Script

Get a ready-to-run command for your deployed model (local or remote):

```bash
distil model invoke <model-id>
```

This outputs a `uv run` command pointing to a client script. Copy and run it directly:

```bash
uv run $PATH_TO_CLIENT --question "Your question here"

# For QA tasks with context
uv run $PATH_TO_CLIENT --question "Your question here" --context "Your context here"
```

### Using the Provided Client Script

The downloaded model includes `model_client.py`. Run it directly:

```bash
python model_client.py --question "Your question here"

# For QA tasks with context
python model_client.py --question "Your question here" --context "Your context here"
```

**Important:** Use the correct system prompt and message formatting when querying your SLM. SLMs are specialized and expect exactly the same format as seen during training. Using a different system prompt or formatting will result in poor performance.

## Remote Deployment

Deploy your model on Distil Labs hosted infrastructure for testing and integration. Remote deployments are not intended for production use -- contact contact@distillabs.ai when you are ready for production.

### Activate a Remote Deployment

```bash
distil model deploy remote <model-id>
```

The CLI provisions your deployment and displays:
- Endpoint URL
- API key
- Client script for querying the model

To output only the client script (useful for piping to a file):

```bash
distil model deploy remote --client-script <model-id>
```

### Deactivate a Remote Deployment

When you are done testing, deactivate to conserve credits:

```bash
distil model deploy remote --deactivate <model-id>
```

### CLI Options Reference

| Option | Description |
|--------|-------------|
| `--client-script` | Output only the client script for the deployment. |
| `--deactivate` | Deactivate a remote deployment. |
| `--output json` | Output results in JSON format. |

## Credits

Remote deployments require credits. All users get $30 of free starting credits. When credits are exhausted, you cannot create new deployments and existing deployments will be deactivated. Contact contact@distillabs.ai when you need more.

## OpenAI-Compatible API

Both local and remote deployments expose an OpenAI-compatible API at the `/v1` endpoint. This means you can integrate your model with any tool or library that supports the OpenAI API format:

```python
from openai import OpenAI

# Local deployment
client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")

# Remote deployment (use the endpoint URL and API key from deploy remote output)
client = OpenAI(base_url="<endpoint-url>/v1", api_key="<your-api-key>")

response = client.chat.completions.create(
    model="model",
    messages=[
        {"role": "system", "content": "Your system prompt"},
        {"role": "user", "content": "Your question"},
    ],
)
print(response.choices[0].message.content)
```
