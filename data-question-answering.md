# Question Answering Data Preparation

Use question answering when the model needs to extract or generate precise answers from text based on specific queries. The input contains both the document and the question.

**Example use cases:**
- Contracts — "What is the termination clause?" or "When does the agreement expire?"
- Invoices — "What's the total amount due?" or "What's the payment deadline?"
- Purchase Orders — "What quantity was ordered?" or "Who is the supplier?"
- Meeting Minutes — "What decisions were made?" or "Who owns the follow-up actions?"
- IT Helpdesk Tickets — "What's the reported issue?" or "What troubleshooting was attempted?"

## Required Files

### 1. job_description.json

```json
{
  "task_description": "Extract the requested information from the provided invoice text. Return only the specific value asked for, without additional explanation. If the information is not found, respond with 'Not found'.",
  "input_description": "Invoice text containing details such as invoice number, date, vendor name, line items, subtotal, tax, and total amount. The question will ask for a specific piece of information from the invoice.",
  "llm_as_a_judge_instructions": "Compare the predicted answer to the reference answer for the given question. Output 'good' if the prediction matches the reference value or is semantically equivalent, otherwise output 'bad'."
}
```

**Fields:**
- `task_description`: Explains what the model should do and output format
- `input_description`: Describes what the input data looks like
- `llm_as_a_judge_instructions` (optional): Evaluation guidance

### 2. train.csv (or train.jsonl)

| Column | Description |
|--------|-------------|
| `question` | The input text or question the model must process |
| `answer` | The expected output or answer |

**JSONL format:**
```json
{"question": "Invoice #1234 from Acme Corp dated 2024-01-15. Items: Widget x10 at $50 each. Subtotal: $500. Tax: $40. Total: $540. What is the total amount?", "answer": "$540"}
{"question": "Invoice #1234 from Acme Corp dated 2024-01-15. Items: Widget x10 at $50 each. Subtotal: $500. Tax: $40. Total: $540. What is the invoice number?", "answer": "1234"}
{"question": "Invoice #1234 from Acme Corp dated 2024-01-15. Items: Widget x10 at $50 each. Subtotal: $500. Tax: $40. Total: $540. Who is the vendor?", "answer": "Acme Corp"}
{"question": "Invoice #5678 from Global Services dated 2024-02-20. Items: Consulting 8hrs at $150/hr. Subtotal: $1200. Tax: $0. Total: $1200. What is the invoice date?", "answer": "2024-02-20"}
```

**CSV format:**

| question | answer |
|----------|--------|
| Invoice #1234 from Acme Corp dated 2024-01-15. Items: Widget x10 at $50 each. Subtotal: $500. Tax: $40. Total: $540. What is the total amount? | $540 |
| Invoice #1234 from Acme Corp dated 2024-01-15. Items: Widget x10 at $50 each. Subtotal: $500. Tax: $40. Total: $540. What is the invoice number? | 1234 |

**Requirements:** Minimum 20 examples

### 3. test.csv (or test.jsonl)

Same format as train data. Used for evaluation, not training.

### 4. config.yaml

The default configuration works well for most cases. You only need to specify the task:

```yaml
base:
  task: question-answering
```

For advanced options (model selection, training parameters, etc.), see `config.md`.

### 5. unstructured.csv (Optional)

Sample documents for synthetic data generation. Single column: `context`

**JSONL format:**
```json
{"context": "Invoice #9012 from Tech Solutions Inc dated 2024-03-10. Items: Software License x1 at $299. Subtotal: $299. Tax: $24. Total: $323."}
{"context": "Invoice #3456 from Office Supplies Co dated 2024-03-15. Items: Paper 10 reams at $8 each, Pens box x5 at $12 each. Subtotal: $140. Tax: $11. Total: $151."}
```

## Upload and Train

```bash
distil model upload-data <model-id> --data ./my-qa-data
distil model run-teacher-evaluation <model-id>
distil model run-training <model-id>
```

## Tips

1. **Include full context in question** — The question field should contain both the source document and the specific question
2. **Precise answers** — Answers should be exact values, not explanations
3. **Diverse question types** — Include different types of questions (what, when, who, how much)
