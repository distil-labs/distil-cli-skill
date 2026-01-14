# Local Deployment

After downloading your trained model with `distil model download <model-id>`, deploy it locally using one of these options.

## Option 1: Ollama

```bash
# Install Ollama from https://ollama.com/

# Create and run model
ollama create my-model -f model/Modelfile
ollama run my-model
```

Query via API:
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
response = client.chat.completions.create(
    model="my-model",
    messages=[{"role": "user", "content": "Your question"}]
)
```

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
| `Modelfile` | For Ollama integration |
| `model_client.py` | Inference script |
| `README.md` | Usage instructions |
