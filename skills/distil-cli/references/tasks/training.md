# Training

Train a small language model (SLM) using knowledge distillation from a large teacher model. Training is split into two commands: generate the synthetic training data, then fine-tune the student on it.

## The Two Steps

```bash
# 1. Synthetic data generation -- 2 credits
distil training-dataset create-from-upload <upload-id>
# Output: Synthetic data generation started. Training Dataset ID: <training-dataset-id>

# 2. Fine-tuning -- several hours
distil slm create-from-training-dataset <training-dataset-id>
# Output: Training started. SLM ID: <slm-id>
```

**Both require explicit user confirmation.** Each is a credit-consuming job, and training credits are hard to refund. In Claude Code, never run either command on your own initiative — show the user the final config, the student/teacher models, and the teacher-evaluation result, then wait for their go-ahead (see the "Confirm Before Training" step in the workflows).

Wait for the training dataset to reach `JOB_SUCCESS` before starting training:

```bash
distil training-dataset status <training-dataset-id> --output json | jq -r '.status'
```

## Inspect the Data Before You Train

Because generation is now a separate step, you can see exactly what the student will learn from before committing to the multi-hour job. This is free and returns in seconds:

```bash
distil training-dataset sample <training-dataset-id>
```

It prints up to 20 rows, deterministically seeded on the dataset ID. Worth doing every time — malformed or off-task generated rows are much cheaper to catch here than after a training run.

`distil training-dataset download <training-dataset-id>` fetches the whole dataset, but it is credit-gated with 0 credits granted by default, so expect a 402 unless credits have been granted for that route. Prefer `sample`.

## Monitor Progress

Training typically takes several hours.

```bash
distil slm status <slm-id>    # status only -- use this when polling
distil slm logs <slm-id>      # logs, for diagnosing a failure
```

For the full status value list, the canonical polling loop, and why `grep` doesn't work, see `references/tasks/polling-jobs.md`.

## Training Stages

What happens across the two commands:

1. **Synthetic data generation** (`training-dataset create-from-upload`) -- the teacher model generates synthetic training data from your problem definition, task description, and provided examples.
2. **Synthetic data validation** (same job) -- the generated data is validated for diversity and quality.
3. **Fine-tuning and evaluation** (`slm create-from-training-dataset`) -- the student model is trained on that data with a loss function aligned to your task, then evaluated on your test set.

## Understanding Training Results

When training completes, `distil slm metrics <slm-id>` reports the base and the tuned student side by side — the comparison that tells you whether fine-tuning actually helped.

```bash
distil slm metrics <slm-id> --output json | jq '.tuned_model_performance'
distil slm metrics <slm-id> --output json | jq '.base_model_performance'
```

The specific metrics depend on your task type:

- **Text generation tasks:** LLM-as-a-Judge, Exact-Match, ROUGE-L
- **Tool calling tasks:** tool_call_equivalence, binary_tool_call, staged_tool_call

For detailed explanations of each metric, see `evaluation-metrics.md`.

### What Makes a Successful Training

- **Comparison to teacher:** Your SLM should achieve performance reasonably close to the teacher model, typically within one standard deviation.
- **Improvement over baseline:** The tuned student should clearly beat `base_model_performance`. If it does not, fine-tuning did not take.
- **Task requirements:** The absolute performance should meet your specific application needs.

## When to Iterate

If the SLM performance is significantly below the teacher model, switch to `workflows/improving-a-model.md` — it covers both iteration cases (teacher eval below thresholds, training results below DEPLOY bar) and the full lever set (job description, data, synthgen/mutations, student/teacher choice, tuning parameters).

## Retrying With Different Tuning Parameters

There is no retune command. To try a different student model or different tuning parameters, reuse the training dataset you already paid to generate rather than regenerating it:

```bash
distil training-dataset download <training-dataset-id> --destination ./dataset
# edit ./dataset/config.yaml -- change base.student_model_name or the tuning section
distil training-dataset create --data ./dataset          # -> new <training-dataset-id>
distil slm create-from-training-dataset <new-training-dataset-id>
```

`download` writes the filenames `create` reads, so the round-trip works without renaming anything. The new dataset's `source` is `direct_upload` rather than `upload`, and it has no generation job behind it, so its status is `JOB_SUCCESS` immediately.

The catch: `training-dataset download` is credit-gated with 0 credits granted by default. If it returns a 402, this path is unavailable and the alternative is a fresh `distil training-dataset create-from-upload <upload-id>` with an updated config on the upload — which pays for generation again.

This is worth doing when:
- Trying a different student model size (e.g., moving from 1B to 3B parameters).
- Adjusting tuning parameters like the number of training epochs.
- Comparing multiple configurations against identical training data.

The confirmation rule still applies: `slm create-from-training-dataset` starts a new credit-consuming run, so get the user's explicit go-ahead first.
