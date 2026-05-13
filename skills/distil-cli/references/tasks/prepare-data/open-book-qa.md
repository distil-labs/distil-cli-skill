# Open Book QA (RAG) Data Preparation

Task type: `question-answering-open-book`

Use open-book QA when the model should answer questions using a provided context passage. The model grounds answers in the given text rather than relying on general knowledge. This is ideal for Retrieval-Augmented Generation (RAG) pipelines.

> **Not supported by `upload-traces`.** Trace processing cannot automatically separate context from question in production logs. If you have RAG-style traces where the retrieved context is already embedded in the user message, use `question-answering` instead and put the full prompt (context + question) in the `question` field. Open Book QA is only for the `upload-data` path with manually prepared datasets.

## When to Pick This Task

- You have (or can retrieve) relevant passages at inference time
- You have chunked documents for a RAG pipeline
- You want answers strictly grounded in provided context

## Example Use Cases

- Customer support systems answering from product documentation
- Legal document analysis and question answering
- Technical documentation assistants
- Knowledge base or FAQ automation
- Research assistants answering from specific papers

## Data Columns

| Column | Description |
|--------|-------------|
| `question` | The question the model must answer |
| `context` | The passage containing information needed to answer |
| `answer` | The expected answer (based on the context) |

## job_description.json

```json
{
  "task_description": "Answer the question using information in the context",
  "llm_as_a_judge_instructions": "Evaluate whether the predicted answer correctly answers the question based on the reference answer. Output 'good' if the predicted answer is semantically equivalent to the reference answer or conveys the same key information, otherwise output 'bad'"
}
```

**Fields:**
- `task_description`: Describes the main task
- `llm_as_a_judge_instructions` (optional): Evaluation guidance. The LLM judge has access to the question, context, reference answer, and predicted answer.

## Train/Test Data Examples

### JSONL format

```json
{"context": "On August 15, 1971, the United States unilaterally pulled out of the Bretton Woods Accord. The US abandoned the Gold Exchange Standard whereby the value of the dollar had been pegged to the price of gold and all other currencies were pegged to the dollar, whose value was left to \"float\" (rise and fall according to market demand).", "question": "What does it mean when currencies are left to \"float?\"", "answer": "rise and fall according to market demand"}
{"context": "The region is home to about 2.5 million insect species, tens of thousands of plants, and some 2,000 birds and mammals. To date, at least 40,000 plant species have been scientifically classified in the region.", "question": "How many species of insects are known in the region?", "answer": "2.5 million"}
{"context": "In the fall quarter of 2014, the University of Chicago enrolled 5,792 students in the College, 3,468 students in its four graduate divisions, 5,984 students in its professional schools, and 15,244 students overall.", "question": "How many students signed up for the university's professional schools in fall 2014?", "answer": "5,984"}
```

### CSV format

| question | context | answer |
|----------|---------|--------|
| What does it mean when currencies are left to "float?" | On August 15, 1971, the United States unilaterally pulled out of the Bretton Woods Accord... | rise and fall according to market demand |
| How many species of insects are known in the region? | The region is home to about 2.5 million insect species... | 2.5 million |

**Requirements:** Minimum 20 examples for both train and test sets.

## config.yaml

```yaml
base:
  task: question-answering-open-book
```

## Unstructured Data (Optional)

Context passages for synthetic data generation. Single column: `context`.

### JSONL format

```json
{"context": "For months each side had been building forward rifle pits and defensive positions. On 5 September, another French bombardment was followed by an assault resulting in the capture of the Malakoff by the French."}
{"context": "The Premier League sells its television rights on a collective basis. The money is divided into three parts: half is divided equally between the clubs; one quarter is awarded on a merit basis based on final league position."}
```

### CSV format

| context |
|---------|
| For months each side had been building forward rifle pits and defensive positions. On 5 September, another French bombardment was followed by an assault resulting in the capture of the Malakoff by the French. |
| The Premier League sells its television rights on a collective basis. The money is divided into three parts: half is divided equally between the clubs; one quarter is awarded on a merit basis based on final league position. |

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

## How Columns Map to Model Input

At training, evaluation, and inference time, the platform combines the `context` and `question` columns into a single user message:

```json
{"role": "user", "content": "Context: <value of context column>\n\nQuestion: <value of question column>"}
```

This means the model is trained to expect input in this `Context: ...\n\nQuestion: ...` format. When querying the deployed model, send the same format:

```python
messages = [{"role": "user", "content": "Context: Refunds are available within 30 days...\n\nQuestion: What is the refund policy?"}]
```

Make sure your RAG pipeline formats the retrieved context this way at inference time.

## Tips

1. **Ground answers in context** -- Answers should only use information from the provided context.
2. **Realistic context lengths** -- Use context sizes similar to what you will retrieve in production.
3. **Varied question types** -- Include factual, procedural, and explanatory questions.
4. **Match your RAG pipeline** -- Use the same chunking strategy in training data as production.
