# Open Book QA (RAG) Data Preparation

Use open-book QA when the model should answer questions using a provided context passage. The model grounds answers in the given text rather than relying on general knowledge. This is ideal for Retrieval-Augmented Generation (RAG) pipelines.

**When to pick this task:**
- You have (or can retrieve) relevant passages at inference time
- You've chunked documents for a RAG pipeline
- You want answers strictly grounded in provided context

**Example use cases:**
- Customer support systems answering from product documentation
- Legal document analysis and question answering
- Technical documentation assistants
- Knowledge base or FAQ automation
- Research assistants answering from specific papers

## Required Files

### 1. job_description.json

```json
{
  "task_description": "Answer the question using information in the context",
  "llm_as_a_judge_instructions": "Evaluate whether the predicted answer correctly answers the question based on the reference answer. Output 'good' if the predicted answer is semantically equivalent to the reference answer or conveys the same key information, otherwise output 'bad'"
}
```

**Fields:**
- `task_description`: Describes the main task
- `llm_as_a_judge_instructions` (optional): Evaluation guidance

### 2. train.csv (or train.jsonl)

| Column | Description |
|--------|-------------|
| `question` | The question the model must answer |
| `context` | The passage containing information needed to answer |
| `answer` | The expected answer (based on the context) |

**JSONL format:**
```json
{"context": "On August 15, 1971, the United States unilaterally pulled out of the Bretton Woods Accord. The US abandoned the Gold Exchange Standard whereby the value of the dollar had been pegged to the price of gold and all other currencies were pegged to the dollar, whose value was left to \"float\" (rise and fall according to market demand).", "question": "What does it mean when currencies are left to \"float?\"", "answer": "rise and fall according to market demand"}
{"context": "The region is home to about 2.5 million insect species, tens of thousands of plants, and some 2,000 birds and mammals. To date, at least 40,000 plant species have been scientifically classified in the region.", "question": "How many species of insects are known in the region?", "answer": "2.5 million"}
{"context": "In the fall quarter of 2014, the University of Chicago enrolled 5,792 students in the College, 3,468 students in its four graduate divisions, 5,984 students in its professional schools, and 15,244 students overall.", "question": "How many students signed up for the university's professional schools in fall 2014?", "answer": "5,984"}
```

**CSV format:**

| question | context | answer |
|----------|---------|--------|
| What does it mean when currencies are left to "float?" | On August 15, 1971, the United States unilaterally pulled out of the Bretton Woods Accord... | rise and fall according to market demand |
| How many species of insects are known in the region? | The region is home to about 2.5 million insect species... | 2.5 million |

**Requirements:** Minimum 20 examples

### 3. test.csv (or test.jsonl)

Same format as train data. Used for evaluation, not training.

### 4. config.yaml

The default configuration works well for most cases. You only need to specify the task:

```yaml
base:
  task: question-answering-open-book
```

For advanced options (model selection, training parameters, etc.), see `config.md`.

### 5. unstructured.csv (Optional)

Context passages for synthetic data generation. Single column: `context`

**JSONL format:**
```json
{"context": "For months each side had been building forward rifle pits and defensive positions. On 5 September, another French bombardment was followed by an assault resulting in the capture of the Malakoff by the French."}
{"context": "The Premier League sells its television rights on a collective basis. The money is divided into three parts: half is divided equally between the clubs; one quarter is awarded on a merit basis based on final league position."}
```

## Upload and Train

```bash
distil model upload-data <model-id> --data ./my-rag-data
distil model run-teacher-evaluation <model-id>
distil model run-training <model-id>
```

## Using the Trained Model

For RAG tasks, provide context when querying:

```bash
python model_client.py --question "What is the refund policy?" --context "Refunds are available within 30 days..."
```

Or via API:
```python
messages = [
    {"role": "user", "content": "Context: Refunds are available within 30 days...\n\nQuestion: What is the refund policy?"}
]
```

## Tips

1. **Ground answers in context** — Answers should only use information from the provided context
2. **Realistic context lengths** — Use context sizes similar to what you'll retrieve in production
3. **Varied question types** — Include factual, procedural, and explanatory questions
4. **Match your RAG pipeline** — Use the same chunking strategy in training data as production
