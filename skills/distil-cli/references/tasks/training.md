# Training

Train a small language model (SLM) using knowledge distillation from a large teacher model. The platform handles synthetic data generation, validation, and fine-tuning automatically.

## Start Training

After teacher evaluation confirms satisfactory performance, start training:

```bash
distil model run-training <model-id>
```

## Monitor Progress

Training typically takes several hours. Check the current status:

```bash
distil model training <model-id>
```

For the full status value list, the canonical polling loop, and why `grep` doesn't work, see `references/tasks/polling-jobs.md`.

## Training Stages

The training process has three stages:

1. **Synthetic data generation** -- The large teacher model generates synthetic training data based on your problem definition, task description, and provided examples.
2. **Synthetic data validation** -- The generated data is validated for diversity and quality to ensure the synthetic set is suitable for training.
3. **Fine-tuning and evaluation** -- The smaller student model is trained on the synthetic data with a loss function aligned with your specific task. After training completes, the student is evaluated on your test set.

## Understanding Training Results

When training completes, `distil model training <model-id>` shows evaluation metrics comparing the trained SLM against the teacher model. The specific metrics depend on your task type:

- **Text generation tasks:** LLM-as-a-Judge, Exact-Match, ROUGE-L, METEOR
- **Tool calling tasks:** tool_call_equivalence, binary_tool_call, staged_tool_call

For detailed explanations of each metric, see `evaluation-metrics.md`.

### What Makes a Successful Training

- **Comparison to teacher:** Your SLM should achieve performance reasonably close to the teacher model, typically within one standard deviation.
- **Task requirements:** The absolute performance should meet your specific application needs.

## When to Iterate

If the SLM performance is significantly below the teacher model, switch to `workflows/improving-a-model.md` — it covers both iteration cases (teacher eval below thresholds, training results below DEPLOY bar) and the full lever set (job description, data, synthgen/mutations, student/teacher choice, tuning parameters, retune).

## Retuning

If you want to try a different student model or different tuning parameters without re-uploading data and re-running the full pipeline, use the retune command. This creates a new model based on the synthetic data already generated from a previous training run.

**Using a tuning parameters file:**

```bash
distil model retune <model-id> \
  --name <name> \
  --student-model <model> \
  --tuning-parameters <file>
```

**Using a full config file** (only the `tuning` section is used):

```bash
distil model retune <model-id> \
  --name <name> \
  --student-model <model> \
  --config <file>
```

| Flag | Alias | Required | Description |
|------|-------|----------|-------------|
| `--name` | `-n` | Yes | Name of the new retuned model to be created. |
| `--student-model` | `-s` | Yes | Student model to use for retuning. |
| `--tuning-parameters` | `-t` | Yes* | Path to tuning parameters file (`.json` or `.yaml`). |
| `--config` | `-c` | Yes* | Path to config file -- only the `tuning` section is used. |

\* Provide either `--tuning-parameters` or `--config`, but not both.

Retuning is useful for:
- Trying a different student model size (e.g., moving from 1B to 3B parameters).
- Adjusting tuning parameters like the number of training epochs.
- Quickly comparing multiple configurations without repeating the full data upload and synthetic generation pipeline.
