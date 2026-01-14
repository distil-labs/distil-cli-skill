# Closed Book QA Data Preparation

Use closed-book QA when the model should learn facts from your unstructured data during training and answer questions using its internal knowledge—no context provided at inference time.

**When to pick this task:**
- You have lots of unstructured data you want to embed into a model
- Users shouldn't need to provide context when asking questions
- A typical RAG setup doesn't work due to retrieval difficulties

**Example use cases:**
- FAQ bots that "know" your product
- Domain-specific knowledge assistants
- Customer support without retrieval
- Information retrieval from embedded knowledge

## Required Files

### 1. job_description.json

```json
{
  "task_description": "The task is to answer the question using your internal knowledge",
  "llm_as_a_judge_instructions": "Evaluate whether the predicted answer correctly answers the question based on the reference answer. Output 'good' if the predicted answer is semantically equivalent to the reference answer or conveys the same key information, otherwise output 'bad'"
}
```

**Fields:**
- `task_description`: Describes the main task (can be simple for closed-book QA)
- `llm_as_a_judge_instructions` (optional): Evaluation guidance

### 2. train.csv (or train.jsonl)

| Column | Description |
|--------|-------------|
| `question` | The question the model must answer |
| `answer` | The expected answer |

**JSONL format:**
```json
{"question": "Where did Sands and Chopin take shelter after the locals in Majorca became inhospitable?", "answer": "a former Carthusian monastery"}
{"question": "When did the Computer Emergency Readiness Team investigate 79 hacking incidents at energy companies?", "answer": "2014"}
{"question": "How is the final quarter of the Premier League's television rights revenue distributed?", "answer": "paid out as facilities fees for games shown on television, with the top clubs receiving the largest shares"}
```

**CSV format:**

| question | answer |
|----------|--------|
| Where did Sands and Chopin take shelter after the locals in Majorca became inhospitable? | a former Carthusian monastery |
| When did the Computer Emergency Readiness Team investigate 79 hacking incidents at energy companies? | 2014 |

**Requirements:** Minimum 20 examples

### 3. test.csv (or test.jsonl)

Same format as train data. Used for evaluation, not training.

### 4. config.yaml

The default configuration works well for most cases. You only need to specify the task:

```yaml
base:
  task: question-answering-closed-book
```

For advanced options (model selection, training parameters, etc.), see `config.md`.

### 5. unstructured.csv (Crucial for this task)

The unstructured data is **crucial** for closed-book QA. This is where you provide the knowledge you want embedded into the model. The system generates QA pairs based on these contexts.

Single column: `context`

**JSONL format:**
```json
{"context": "In June 1837 Chopin visited London incognito in the company of the piano manufacturer Camille Pleyel. The two spent a miserable winter on Majorca (8 November 1838 to 13 February 1839). After discovering that the couple were not married, the deeply traditional Catholic people of Majorca became inhospitable, making accommodation difficult to find. This compelled the group to take lodgings in a former Carthusian monastery in Valldemossa."}
{"context": "The Premier League sells its television rights on a collective basis. The money is divided into three parts: half is divided equally between the clubs; one quarter is awarded on a merit basis based on final league position; the final quarter is paid out as facilities fees for games that are shown on television, with the top clubs generally receiving the largest shares."}
{"context": "Computers control functions at many utilities, including coordination of telecommunications, the power grid, and nuclear power plants. In 2014, the Computer Emergency Readiness Team, a division of the Department of Homeland Security, investigated 79 hacking incidents at energy companies."}
```

## Upload and Train

```bash
distil model upload-data <model-id> --data ./my-closed-book-data
distil model run-teacher-evaluation <model-id>
distil model run-training <model-id>
```

## Using the Trained Model

For closed-book tasks, just ask questions directly—no context needed:

```bash
python model_client.py --question "What is your refund policy?"
```

Or via API:
```python
messages = [
    {"role": "user", "content": "What is your refund policy?"}
]
```

## Tips

1. **Comprehensive unstructured data** — Include all facts you want the model to know
2. **Consistent information** — Avoid contradictions in your data
3. **Unstructured data is crucial** — This is how knowledge gets embedded into the model
4. **Consider scope** — Closed-book models work best for bounded domains with clear factual answers
5. **Quality over quantity in train data** — The train examples show the style; unstructured data provides the knowledge
