# Local Deployment

After downloading your trained model with `distil model download <model-id>`, deploy it locally using one of these options.

## Option 1: Distil CLI (Recommended)

Deploy your trained model locally using the built-in `distil model deploy` command, which uses `llama-cpp` as the inference backend:

```bash
distil model deploy local <model-id>
```

This downloads the model and starts a local llama-server on port 8000. Customize the port and enable server logs:

```bash
distil model deploy local --port 9000 --logs <model-id>
```

Once running, the model is available via the OpenAI-compatible API at `http://localhost:<port>/v1`.

To get the command to invoke your locally deployed model:

```bash
distil model invoke <model-id>
```

This outputs a ready-to-run command using [uv](https://docs.astral.sh/uv/). Copy and run it directly:

```bash
uv run $PATH_TO_CLIENT --question "Your question here"

# For QA tasks with context
uv run $PATH_TO_CLIENT --question "Your question here" --context "Your context here"
```

| Option | Description |
|--------|-------------|
| `--port <port>` | Port number for local llama-server (default: 8000) |
| `--logs` | Show llama-server logs during local deployment |
| `--output json` | Output results in JSON format |

**Note:** Local deployment requires [llama-cpp](https://github.com/ggerganov/llama.cpp) to be installed on your machine.

## Option 2: vLLM

```bash
pip install vllm openai
vllm serve model --api-key EMPTY
```

Query via API:
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")
response = client.chat.completions.create(
    model="model",
    messages=[{"role": "user", "content": "Your question"}]
)
```

## Using the Provided Client

The downloaded model includes a `model_client.py` script:

```bash
python model_client.py --question "Your question here"

# For QA tasks with context
python model_client.py --question "Your question" --context "Your context"
```

## Downloaded Model Contents

When you run `distil model download <model-id>`, you get:

| File/Directory | Description |
|----------------|-------------|
| `model/` | Model weights |
| `model-adapters/` | LoRA adapters |
| `model_client.py` | Inference script |
| `README.md` | Usage instructions |