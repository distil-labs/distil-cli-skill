# Classification Data Preparation

Use classification when the model needs to analyze input text and assign it to one category from a fixed set of options. This task type is effective when you need deterministic categorization rather than open-ended generation.

**Example use cases:**
- Intent detection for customer service queries
- Content moderation (toxic/safe, spam/not spam)
- Sentiment analysis (positive/negative/neutral)
- Topic categorization for knowledge bases
- Triaging support tickets by department

## Required Files

### 1. job_description.json

```json
{
  "task_description": "Classify the bank customer service requests into one of the provided classes",
  "classes_description": {
    "balance_not_updated_after_bank_transfer": "Requests about a completed bank transfer not yet reflected in the account balance.",
    "balance_not_updated_after_cheque_or_cash_deposit": "Requests regarding a recent cheque or cash deposit not showing up in the available account balance.",
    "card_payment_fee_charged": "Requests questioning an unexpected or additional fee charged for making a payment with a card.",
    "cash_withdrawal_charge": "Requests related to being charged a fee for withdrawing cash from an ATM.",
    "declined_cash_withdrawal": "Requests about attempting to withdraw cash from an ATM but having the transaction declined.",
    "direct_debit_payment_not_recognised": "Requests regarding an unauthorized direct debit payment charged to the account."
  }
}
```

**Fields:**
- `task_description`: Describes the main classification task
- `classes_description`: Map from class names to their descriptions

### 2. train.csv (or train.jsonl)

| Column | Description |
|--------|-------------|
| `question` | The input text to classify |
| `answer` | The class label |

**JSONL format:**
```json
{"question": "Why is there a fee for getting cash?", "answer": "cash_withdrawal_charge"}
{"question": "I was declined when I tried to take out cash!", "answer": "declined_cash_withdrawal"}
{"question": "I deposited some money, but the balance has not changed.", "answer": "balance_not_updated_after_cheque_or_cash_deposit"}
{"question": "It has been a couple of hours but I do not see my balance updated, can you help?", "answer": "balance_not_updated_after_bank_transfer"}
{"question": "There is a payment showing on my app that I didn't do. Will you please cancel this payment?", "answer": "direct_debit_payment_not_recognised"}
{"question": "How do I know which payments I make will have additional fees?", "answer": "card_payment_fee_charged"}
```

**CSV format:**

| question | answer |
|----------|--------|
| Why is there a fee for getting cash? | cash_withdrawal_charge |
| I was declined when I tried to take out cash! | declined_cash_withdrawal |
| I deposited some money, but the balance has not changed. | balance_not_updated_after_cheque_or_cash_deposit |

**Requirements:** Minimum 20 examples. Include examples for all classes.

### 3. test.csv (or test.jsonl)

Same format as train data. Used for evaluation, not training.

### 4. config.yaml

The default configuration works well for most cases. You only need to specify the task:

```yaml
base:
  task: classification
```

For advanced options (model selection, training parameters, etc.), see `config.md`.

### 5. unstructured.csv (Optional)

Unlabelled examples or domain documentation to guide synthetic data generation. Single column: `context`

**JSONL format:**
```json
{"context": "Canceling my order is what I need to do right now."}
{"context": "I swear that there are 2 payments on the app that I didn't make. Could my card be stolen?"}
{"context": "Too many charges on my card, how do I go about fixing that?"}
{"context": "I got less cash than what I specified at the ATM."}
{"context": "Why is my last cheque deposit taking so long?"}
```

## Upload and Train

```bash
distil model upload-data <model-id> --data ./my-classification-data
distil model run-teacher-evaluation <model-id>
distil model run-training <model-id>
```

## Tips

1. **Balance your classes** — Include similar numbers of examples per class
2. **Clear class boundaries** — Ensure classes don't overlap significantly
3. **Descriptive class names** — Use clear names and descriptions in `classes_description`
4. **Representative examples** — Cover the variety of inputs you expect in production
