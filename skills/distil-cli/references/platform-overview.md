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

**Models.** Each experiment is tracked as a model, identified by a human-readable name and a UUID. Create a model with `distil model create <name>`. Use the returned model ID in all subsequent commands.

**Uploads.** Training data is attached to a model via an upload. Use `distil model upload-data` for structured datasets or `distil model upload-traces` for production traces. A model can have multiple uploads as you iterate on the data.

**Trace processing.** When you upload production traces, the platform runs an automated pipeline: filtering traces for relevance, relabelling via a committee of teacher models, and splitting into training and test sets. This transforms raw production logs into high-quality structured training data. Use `distil model reprocess-traces` to iterate on processing parameters without re-uploading.

**Teacher evaluation.** Before training, validate that the teacher model can solve the task. Run with `distil model run-teacher-evaluation <model-id>`. High teacher accuracy predicts good student performance. Low accuracy signals that the task description or data needs revision.

**Training.** The full distillation pipeline: synthetic data generation, validation, and student fine-tuning. Start with `distil model run-training <model-id>`. Training takes several hours.

**Deployment.** After training, download the model with `distil model download <model-id>` and deploy it locally (`distil model deploy local <model-id>`) or to distil-managed remote infrastructure (`distil model deploy remote <model-id>`).

## Supported Task Types

The platform supports six task types. For model compatibility constraints (which students and teachers work with which task), see `references/model-catalog.md`.

**Question Answering** -- extract or generate precise answers from text based on specific queries. The most general task type. Use for QA, text transformations, or any task that takes text input and produces text output. Examples: contract clause extraction, document summarization, data reformatting.

**Classification** -- assign text to one category from a fixed set. Use when you need deterministic categorization, not open-ended generation. Examples: intent detection, sentiment analysis, content moderation, ticket triage.

**Tool Calling** -- map natural language to structured function calls with correct parameters. Use when routing user requests to backend APIs or services. Examples: voice assistant commands, chatbot-to-CRM routing, natural language to API calls.

**Multi-Turn Tool Calling** -- generate function calls within a conversational context, maintaining state across multiple turns. Examples: file system assistants, database query interfaces, DevOps chatbots.

**Open Book QA (RAG)** -- answer questions using provided context passages. Only use this if you already have a well-structured knowledge database with context chunks. The model expects retrieved context at inference time. Examples: customer support from product docs, legal document analysis, technical documentation assistants.

**Closed Book QA** -- answer questions from knowledge learned during training. Provide a knowledge database and the model learns from it during training; no context is needed at inference. Examples: FAQ bots, domain-specific knowledge assistants.

## Two Data Paths

**Structured dataset upload.** Prepare labeled files manually (`job_description.json`, `train.csv`, `test.csv`, `config.yaml`, optional `unstructured.csv`) and upload with:

```bash
distil model upload-data <model-id> --data ./my-data-folder
```

**Production traces.** If you have production logs from real LLM interactions (e.g., Langfuse or OpenAI messages format), upload them directly. The platform automatically processes traces into training and test data:

```bash
distil model upload-traces <model-id> --data ./my-traces-folder
```

To reprocess previously uploaded traces with different parameters:

```bash
distil model reprocess-traces <model-id> --trace-processing-config new-config.yaml
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
    | 4. Multi-turn handling        |   convert_to_single_turn: true
    |    true  -> split by          |     -> one example per assistant
    |             assistant turn    |        turn (use for single-turn
    |    false -> keep whole        |        tasks)
    |             conversation,     |   convert_to_single_turn: false
    |             rewrite as one    |     -> preserve conversation
    |             example           |         (required for
    |                               |          multi-turn-tool-calling-
    |                               |          closed-book)
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

Remote deployments on distil-managed infrastructure require credits. All users receive $30 of free starting credits. Deactivate deployments when not in use to conserve credits:

```bash
distil model deploy remote --deactivate <model-id>
```

Contact [contact@distillabs.ai](mailto:contact@distillabs.ai) for additional credits. Remote deployments are intended for testing; contact Distil Labs to set up production deployments.
