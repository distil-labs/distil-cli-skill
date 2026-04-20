# Task Selection Guide

Choosing the right task type is the most important decision in the Distil Labs workflow. This guide covers all six supported task types with use cases, tradeoffs, and a decision flowchart.

---

## Quick Decision Table

| If you need to...                                        | Choose                            | Config value                             |
|----------------------------------------------------------|-----------------------------------|------------------------------------------|
| Generate text answers from input text                    | Question Answering                | `question-answering`                     |
| Assign text to one category from a fixed set             | Classification                    | `classification`                         |
| Map natural language to function/API calls               | Tool Calling                      | `tool-calling-closed-book`               |
| Generate function calls in multi-turn conversations      | Multi-Turn Tool Calling           | `multi-turn-tool-calling-closed-book`    |
| Answer questions using provided context passages (RAG)   | Open Book QA                      | `question-answering-open-book`           |
| Answer questions from knowledge learned during training  | Closed Book QA                    | `question-answering-closed-book`         |

---

## Decision Flowchart

```
Does the model need to produce structured function/API calls?
  |
  +-- YES --> Is the interaction multi-turn (conversational)?
  |             |
  |             +-- YES --> Multi-Turn Tool Calling
  |             |           (multi-turn-tool-calling-closed-book)
  |             |
  |             +-- NO  --> Tool Calling
  |                         (tool-calling-closed-book)
  |
  +-- NO  --> Does the model assign text to a fixed set of categories?
                |
                +-- YES --> Classification
                |           (classification)
                |
                +-- NO  --> Does the model need external context at inference time?
                              |
                              +-- YES --> Do you have a RAG pipeline / knowledge database?
                              |             |
                              |             +-- YES --> Open Book QA
                              |             |           (question-answering-open-book)
                              |             |
                              |             +-- NO  --> Question Answering
                              |                         (question-answering)
                              |
                              +-- NO  --> Should knowledge be baked into the model?
                                            |
                                            +-- YES --> Closed Book QA
                                            |           (question-answering-closed-book)
                                            |
                                            +-- NO  --> Question Answering
                                                        (question-answering)
```

---

## When the Flowchart Isn't Enough (Use Judgment)

The flowchart handles clear-cut cases. These common ambiguities require judgment:

- **Hybrid tasks** (e.g., "extract data and classify it") — help the user decompose into two models or pick the dominant pattern based on what matters most in production.
- **Open Book vs. Closed Book QA** — ask about infrastructure. If they have a working retrieval pipeline, use Open Book. If they don't, or retrieval is unreliable, use Closed Book.
- **"Tool calling" that's really classification** — if the user's "tools" are just category labels with no meaningful parameters (e.g., `route_to_billing()`, `route_to_support()`), suggest classification instead. Tool calling adds complexity that classification doesn't need.
- **Question Answering as the catch-all** — if nothing else fits, QA is the right default. Don't force a task type that doesn't match.

---

## Task Type Details

### 1. Question Answering (`question-answering`)

The most general task type. The model takes text input and produces text output -- extracting information, generating answers, or transforming text.

**When to use:**
- You need to extract specific facts from documents
- You need text transformations (reformatting, summarizing, rewriting)
- Your task is "text in, text out" and does not fit another category

**Example use cases:**
- Document extraction: "What is the termination clause?" from contracts
- Invoice parsing: "What is the total amount due?"
- Meeting note summaries: "What decisions were made?"
- IT helpdesk: "What troubleshooting was attempted?"
- Data reformatting: "Reformat this data as JSON"

**Data format:** `question` and `answer` columns in train/test CSV or JSONL.

---

### 2. Classification (`classification`)

Assigns input text to exactly one category from a predefined set. Produces deterministic categorization, not open-ended text.

**When to use:**
- You have a fixed, known set of output categories
- You need consistent, deterministic assignment
- The output is a label, not free-form text

**Example use cases:**
- Intent detection for customer service queries
- Sentiment analysis (positive / negative / neutral)
- Content moderation (toxic / safe, spam / not spam)
- Topic categorization for knowledge bases
- Support ticket triage by department

**Data format:** `question` and `answer` columns, where `answer` is one of the predefined class labels. Classes are defined in `job_description.json` via `classes_description`.

---

### 3. Tool Calling (`tool-calling-closed-book`)

Maps natural language to structured function calls with correct parameters. The model learns to dispatch user intents to specific backend functions or APIs.

**When to use:**
- You have a fixed set of tools/APIs the model should invoke
- Each input independently maps to one function call
- You need structured, schema-compliant outputs
- Tool selection logic should be memorized during training

**Example use cases:**
- Voice assistants: spoken commands to smart home APIs
- Chatbot actions: user intents to CRM/database operations
- API routing: natural language to backend service calls
- Workflow automation: requests to appropriate microservices
- Command interfaces: user input to system commands

**Model constraints:** Only Qwen3 and Llama 3-family student models. All teacher models work. See `references/model-catalog.md`.

**Data format:** `question` (plain text) and `answer` (JSON string of the tool call with `name` and `parameters`). Tools are defined in `job_description.json`.

---

### 4. Multi-Turn Tool Calling (`multi-turn-tool-calling-closed-book`)

Generates function calls within a conversational context. Unlike single-turn tool calling, this takes a full conversation history and generates the next appropriate function call based on accumulated context.

**When to use:**
- Your application requires conversational interactions with tool invocations
- Users issue sequences of related commands that build on each other
- The model must understand context from previous turns
- You want dialogue-based interfaces to APIs or services

**Example use cases:**
- File system assistants: navigate directories and manage files through conversation
- Database query builders: build complex queries through iterative refinement
- DevOps chatbots: execute sequences of infrastructure commands conversationally
- Smart home controllers: chain home automation commands naturally
- Customer service bots: handle multi-step service requests in dialogue
- IDE assistants: execute code operations through conversational commands

**Model constraints:** Students must be Qwen3 or Llama 3-family. Teachers are restricted to a 10-model allowlist (GPT OSS 120B and 120B-thinking, Llama 3.1 405B, Qwen3 235B, DeepSeek V3.2 and V3.2-thinking, GLM 5 and GLM 5-thinking, Kimi K2-thinking, MiniMax M2-thinking). See `references/model-catalog.md` for the exact config strings.

**Data format:** `question` (JSON array of conversation turns with alternating user/assistant messages) and `answer` (JSON string of the next tool call). Tools are defined in `job_description.json`.

---

### 5. Open Book QA (`question-answering-open-book`)

Answers questions using provided context passages. The model generates answers grounded strictly in the text provided at inference time. This is the natural fit for RAG (Retrieval-Augmented Generation) pipelines.

**When to use:**
- You already have a RAG pipeline with good retrieval
- You have a well-structured knowledge database with context chunks
- The model should answer strictly from provided context, not from general knowledge
- You want grounded, verifiable answers

**When NOT to use:**
- You do not have a knowledge database yet
- Your context chunks are poorly structured
- You want the model to answer from its own learned knowledge (use Closed Book QA instead)

**Example use cases:**
- Customer support from product documentation
- Legal document analysis and question answering
- Technical documentation assistants
- Knowledge base or FAQ automation
- Research assistants answering from specific papers

**Data format:** `question`, `context`, and `answer` columns. The `context` column provides the passage the model should use.

---

### 6. Closed Book QA (`question-answering-closed-book`)

The model learns facts and knowledge from unstructured data during training. At inference time, the user asks questions and the model answers from its internal knowledge -- no context is provided.

**When to use:**
- You have a large amount of unstructured domain data
- You want knowledge "baked into" the model
- Users should not need to provide context when asking questions
- A RAG setup does not work for you (difficult retrieval, latency constraints, etc.)

**Example use cases:**
- FAQ bots that answer from learned product knowledge
- Domain-specific knowledge assistants
- Information retrieval from arbitrary contexts
- Customer support without a retrieval pipeline

**Data format:** `question` and `answer` columns for train/test data, plus a required `unstructured.csv` (or `.jsonl`) containing the domain text the model should learn from.

---

## Choosing Between Open Book QA and Closed Book QA

This is the most common point of confusion. Here is the key distinction:

| Aspect                  | Open Book QA (RAG)                          | Closed Book QA                              |
|-------------------------|---------------------------------------------|---------------------------------------------|
| Context at inference    | Required -- provided by your retrieval system | Not needed -- knowledge is in the model    |
| Knowledge source        | Retrieved passages at query time            | Unstructured data absorbed during training  |
| Prerequisite            | Working RAG pipeline with good retrieval    | Domain text corpus                          |
| Best for                | Large, evolving knowledge bases             | Static domain knowledge, FAQ-style bots     |
| Updatability            | Update retrieval index, no retraining       | Retrain to update knowledge                 |

---

## Model Compatibility

See `references/model-catalog.md` for the authoritative compatibility matrix (student families per task, teacher constraints for multi-turn tool calling, hard rules). Do not duplicate the matrix here — it drifts.
