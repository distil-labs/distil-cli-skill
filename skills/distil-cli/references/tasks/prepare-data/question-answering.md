# Question Answering Data Preparation

Task type: `question-answering`

Use question answering when the model needs to extract or generate precise answers from text based on specific queries. The input contains both the document and the question.

## Example Use Cases

- **Contracts** -- "What is the termination clause?" or "When does the agreement expire?"
- **Invoices** -- "What's the total amount due?" or "What's the payment deadline?"
- **Purchase Orders** -- "What quantity was ordered?" or "Who is the supplier?"
- **Meeting Minutes** -- "What decisions were made?" or "Who owns the follow-up actions?"
- **IT Helpdesk Tickets** -- "What's the reported issue?" or "What troubleshooting was attempted?"

## Data Format

Each example is a `messages` conversation with one `user` turn (the input) and one `assistant` turn (the expected answer).

| Field | Description |
|-------|-------------|
| `messages` | A `user` turn holding the input/question and an `assistant` turn holding the expected answer |

## job_description.json

```json
{
  "task_description": "Extract the requested information from the provided invoice text. Return only the specific value asked for, without additional explanation. If the information is not found, respond with 'Not found'.",
  "input_description": "Invoice text containing details such as invoice number, date, vendor name, line items, subtotal, tax, and total amount. The question will ask for a specific piece of information from the invoice.",
  "llm_as_a_judge_instructions": "Compare the predicted answer to the reference answer for the given question. Output 'good' if the prediction matches the reference value or is semantically equivalent, otherwise output 'bad'."
}
```

**Fields:**
- `task_description` (required): Explains what the model should do and output format
- `input_description` (required): Describes what the input data looks like. The platform validates this field is present and will reject uploads without it.
- `llm_as_a_judge_instructions` (optional): Evaluation guidance

## Train/Test Data Examples

```json
{"messages": [{"role": "user", "content": "Invoice #1234 from Acme Corp dated 2024-01-15. Items: Widget x10 at $50 each. Subtotal: $500. Tax: $40. Total: $540. What is the total amount?"}, {"role": "assistant", "content": "$540"}]}
{"messages": [{"role": "user", "content": "Invoice #1234 from Acme Corp dated 2024-01-15. Items: Widget x10 at $50 each. Subtotal: $500. Tax: $40. Total: $540. What is the invoice number?"}, {"role": "assistant", "content": "1234"}]}
{"messages": [{"role": "user", "content": "Invoice #1234 from Acme Corp dated 2024-01-15. Items: Widget x10 at $50 each. Subtotal: $500. Tax: $40. Total: $540. Who is the vendor?"}, {"role": "assistant", "content": "Acme Corp"}]}
{"messages": [{"role": "user", "content": "Invoice #5678 from Global Services dated 2024-02-20. Items: Consulting 8hrs at $150/hr. Subtotal: $1200. Tax: $0. Total: $1200. What is the invoice date?"}, {"role": "assistant", "content": "2024-02-20"}]}
```

**Requirements:** Minimum 20 examples for both train and test sets.

## config.yaml

```yaml
base:
  task: question-answering
```

## Unstructured Data (Optional)

Sample documents for synthetic data generation. Single field: `context`.

```json
{"context": "Invoice #9012 from Tech Solutions Inc dated 2024-03-10. Items: Software License x1 at $299. Subtotal: $299. Tax: $24. Total: $323."}
{"context": "Invoice #3456 from Office Supplies Co dated 2024-03-15. Items: Paper 10 reams at $8 each, Pens box x5 at $12 each. Subtotal: $140. Tax: $11. Total: $151."}
```

## How the Data Maps to Model Input

Your examples are already in the format the model sees: the `user` turn is the input, the `assistant` turn is the expected output. The `user` content should contain **everything** the model needs to answer — both the source document/context and the specific question, exactly as you would send it in production.

When querying the deployed model, send the same shape:
```python
messages = [{"role": "user", "content": "Invoice #1234 from Acme Corp... What is the total amount?"}]
```

## Tips

1. **Include full context in the user turn** -- The `user` content should contain both the source document and the specific question.
2. **Precise answers** -- Answers should be exact values, not explanations.
3. **Diverse question types** -- Include different types of questions (what, when, who, how much).
4. **Representative documents** -- Use input documents that resemble what the model will encounter in production.
