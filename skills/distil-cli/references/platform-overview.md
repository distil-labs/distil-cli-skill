# Platform Overview

## What Is Distil Labs

Distil Labs is a fully managed platform that transforms your existing LLM system prompt, paired with either a minimal dataset (10-50 examples) or production logs from your running agent, into a production-ready small language model (SLM) in under 12 hours, with no ML expertise required. Under the hood, the platform uses large language models as "teachers" to generate high-quality synthetic training data from these inputs, validates and curates that data through automated filters, then fine-tunes compact "student" models via knowledge distillation. The resulting SLMs are 50-150x smaller than cloud LLMs and dramatically cheaper to run, yet they often match or exceed the original teacher models on the specific tasks they are trained for.

## Value Proposition

- **50-150x smaller models** with comparable accuracy to large language models.
- **Minimal data** -- as few as 10-50 labeled examples, or just production traces from your running agent.
- **No ML expertise needed** -- the platform handles synthetic data generation, training, and optimization.
- **Task-specific** -- each trained model is specialized for a single, well-defined task, yielding higher accuracy and lower latency than general-purpose models.

## How It Works

The platform uses knowledge distillation to transfer capabilities from a large "teacher" model into a small "student" model:

1. **Synthetic data generation** -- the teacher model generates synthetic training data based on the task description and provided examples.
2. **Synthetic data validation** -- generated data is validated for diversity and quality.
3. **Knowledge transfer** -- the student model is trained on the synthetic data with a task-specific loss function, learning to emulate the teacher's capabilities at a fraction of the size.

## Key Concepts

Every stage of the platform is its own entity with its own UUID, and each stage consumes the ID the previous one printed.

```
prepared traces -> upload -> teacher evaluation      (feasibility check, a side branch)
                   upload -> training dataset -> SLM -> deployment
```

The links are stored on the platform, not just in your notes: a deployment records its `slm_id`, an SLM its `training_dataset_id`, a training dataset and a teacher evaluation their `upload_id`, and an upload its `prepared_traces_id`. So the full lineage behind any entity is recoverable from that entity alone — see `references/cli-reference.md` (`### Tracing the Chain`).

**Prepared traces.** Raw production logs uploaded with `distil traces upload`, identified by a UUID and managed with `distil traces list` / `show` / `download`. They are the input to trace processing, not training data themselves. Optional — only the traces path uses them.

**Trace processing.** `distil upload create-from-traces <traces-id>` runs an automated pipeline over prepared traces: filtering traces for relevance, relabelling via a committee of teacher models, and splitting into training and test sets. This transforms raw production logs into high-quality structured training data, producing an upload. Re-run it against the same traces ID with a different `--config` to iterate on processing parameters without re-uploading the trace files.

**Uploads.** An upload is a set of training and test data, identified by a UUID. It is the entry point to the pipeline: teacher evaluation and synthetic data generation both read an upload. Create one from local files with `distil upload create --data <dir>`, or from prepared traces with `distil upload create-from-traces`. Inspect any upload with `distil upload list` / `show` / `status`.

**Teacher evaluation.** Before spending credits on training, validate that the teacher model can solve the task: `distil teacher-evaluation create-from-upload <upload-id>`, then `distil teacher-evaluation metrics <teacher-evaluation-id>`. High teacher accuracy predicts good student performance. Low accuracy signals that the task description or data needs revision. This is a side branch — it gates the decision to train, but nothing downstream consumes its ID.

**Training datasets.** An upload's data with synthetic training examples generated for it, identified by a UUID. Create one with `distil training-dataset create-from-upload <upload-id>` (2 credits). This is a **required stage** — training reads a training dataset, not an upload. The useful side effect is that `distil training-dataset sample <training-dataset-id>` shows you the generated rows for free, so you can see what the model will learn from before committing to the multi-hour training job.

**Training.** `distil slm create-from-training-dataset <training-dataset-id>` fine-tunes the student on the generated data. Takes several hours and burns credits that are hard to refund.

**SLMs.** A trained small language model — the model tarball plus the config it came from — identified by a UUID and managed with `distil slm list` / `show` / `status` / `logs` / `metrics` / `download`. `distil slm metrics <slm-id>` is the one place that shows the base and the tuned student side by side, which is how you judge whether fine-tuning actually helped. You can also bring your own with `distil slm create`.

**Deployments.** `distil deployment create-from-slm <slm-id>` serves an SLM from Distil Labs infrastructure behind an OpenAI-compatible endpoint; `distil deployment endpoint <deployment-id>` prints its URL and API key, and `distil deployment delete <deployment-id>` shuts it down. To serve the model yourself instead, `distil slm download <slm-id>` writes `model.tar` and `config.yaml` for you to run under vLLM or llama-cpp.

## Supported Task Types

The platform supports six task types. For model compatibility constraints (which students and teachers work with which task), see `references/model-catalog.md`.

**Question Answering** -- extract or generate precise answers from text based on specific queries. The most general task type. Use for QA, text transformations, or any task that takes text input and produces text output. Examples: contract clause extraction, document summarization, data reformatting.

**Classification** -- assign text to one category from a fixed set. Use when you need deterministic categorization, not open-ended generation. Examples: intent detection, sentiment analysis, content moderation, ticket triage.

**Tool Calling** -- map natural language to structured function calls with correct parameters. Use when routing user requests to backend APIs or services. Examples: voice assistant commands, chatbot-to-CRM routing, natural language to API calls.

**Multi-Turn Tool Calling** -- generate function calls within a conversational context, maintaining state across multiple turns. Examples: file system assistants, database query interfaces, DevOps chatbots.

**Open Book QA (RAG)** -- answer questions using provided context passages. Only use this if you already have a well-structured knowledge database with context chunks. The model expects retrieved context at inference time. Examples: customer support from product docs, legal document analysis, technical documentation assistants.

**Closed Book QA** -- answer questions from knowledge learned during training. Provide a knowledge database and the model learns from it during training; no context is needed at inference. Examples: FAQ bots, domain-specific knowledge assistants.

## Two Data Paths

**Structured dataset upload.** Prepare labeled files manually (`job_description.json`, `train.jsonl`, `test.jsonl`, `config.yaml`, optional `unstructured.jsonl`) and upload with:

```bash
distil upload create --data ./my-data-folder           # -> <upload-id>
```

**Production traces.** If you have production logs from real LLM interactions (e.g., Langfuse or OpenAI messages format), let the platform derive the dataset for you. Upload the trace files and process them:

```bash
distil traces upload --data ./my-traces-folder         # -> <traces-id>
distil upload create-from-traces <traces-id>           # -> <upload-id>
distil upload status <upload-id>                       # poll until JOB_SUCCESS
```

Both paths end at an `<upload-id>`, so everything downstream is identical. There is no need to download processed traces and re-upload them — do that only when you want to inspect or hand-edit what trace processing produced:

```bash
distil upload download <upload-id> --destination ./processed
```

To reprocess the same traces with different parameters, re-run `create-from-traces` with a new config -- no need to re-upload the trace files:

```bash
distil upload create-from-traces <traces-id> --config new-config.yaml
```

## Trace Processing Pipeline

Uploaded traces are transformed into train/test data by a four-stage pipeline. The diagram shows each stage, what it does, and the `trace_processing` parameters that control it. See `references/tasks/upload-and-process-traces.md` for step-by-step usage and `references/configuration.md` for full parameter semantics.

```
    +-------------------------------+
    |       traces.jsonl            |   raw traces
    |  (+ job_description.json)     |
    +--------------+----------------+
                   |
                   v
    +-------------------------------+
    | 1. Relevance filtering        |   teacher_model_name scores
    |    drop traces not relevant   |   each trace; irrelevant ones
    |    to the task                |   are discarded
    +--------------+----------------+   compress_job_description
                   |                    remove_system_prompt_from_traces
                   v
    +-------------------------------+
    | 2. Relabelling                |   relabel: true
    |    committee rewrites labels, |   relabelling_committee_models
    |    teacher picks the best     |   (if relabel: false, original
    |                               |    labels are kept)
    +--------------+----------------+
                   |
                   v
    +-------------------------------+
    | 3. Split into train + test    |   num_traces_as_training_base
    |    leftover -> unstructured   |   num_traces_as_testing_base
    |    context                    |   max_unstructured
    |    (skipped for test if       |   (see "Test Set Behavior"
    |     user provided test.jsonl) |    in upload-and-process-traces)
    +--------------+----------------+
                   |
                   v
    +-------------------------------+
    | 4. Validation & fixing        |   every trace is processed as a
    |    validate against the task, |   multi-turn conversation and
    |    LLM-repair where needed;   |   rewritten as a whole (a simple
    |    conversations kept whole   |   exchange is just a 2-turn convo)
    +--------------+----------------+
                   |
                   v
    +-------------------------------+
    | Processed train + test        |   min_generated_examples gate:
    | + unstructured context        |   errors if fewer produced
    +-------------------------------+   (keep low enough to clear
                                        filtering + relabelling drops)
```

## Credit System

Credits are consumed by synthetic data generation (2 credits per training dataset), training, and hosted deployments while they run. All users receive $30 of free starting credits. Shut deployments down when not in use to conserve credits:

```bash
distil deployment delete <deployment-id>      # alias: distil deployment shutdown
```

Contact [contact@distillabs.ai](mailto:contact@distillabs.ai) for additional credits. Hosted deployments are intended for testing; contact Distil Labs to set up production deployments.
