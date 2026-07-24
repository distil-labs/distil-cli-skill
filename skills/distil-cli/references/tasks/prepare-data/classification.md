# Classification Data Preparation

Task type: `classification`

Use classification when the model needs to analyze input text and assign it to one category from a fixed set of options. This task type is effective when you need deterministic categorization rather than open-ended generation.

## Example Use Cases

- Intent detection for customer service queries
- Content moderation (toxic/safe, spam/not spam)
- Sentiment analysis (positive/negative/neutral)
- Topic categorization for knowledge bases
- Triaging support tickets by department

## Data Format

Each example is a `messages` conversation with one `user` turn (the input to classify) and one `assistant` turn (the class label).

| Field | Description |
|-------|-------------|
| `messages` | A `user` turn holding the input text and an `assistant` turn holding the class label |

## job_description.json

Classification requires two fields: `task_description` and `classes_description`.

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
- `classes_description`: Map from class names to their descriptions. Every class the model should predict must appear in this map.

## Train/Test Data Examples

```json
{"messages": [{"role": "user", "content": "Why is there a fee for getting cash?"}, {"role": "assistant", "content": "cash_withdrawal_charge"}]}
{"messages": [{"role": "user", "content": "I was declined when I tried to take out cash!"}, {"role": "assistant", "content": "declined_cash_withdrawal"}]}
{"messages": [{"role": "user", "content": "I deposited some money, but the balance has not changed."}, {"role": "assistant", "content": "balance_not_updated_after_cheque_or_cash_deposit"}]}
{"messages": [{"role": "user", "content": "It has been a couple of hours but I do not see my balance updated, can you help?"}, {"role": "assistant", "content": "balance_not_updated_after_bank_transfer"}]}
{"messages": [{"role": "user", "content": "There is a payment showing on my app that I didn't do. Will you please cancel this payment?"}, {"role": "assistant", "content": "direct_debit_payment_not_recognised"}]}
{"messages": [{"role": "user", "content": "How do I know which payments I make will have additional fees?"}, {"role": "assistant", "content": "card_payment_fee_charged"}]}
```

**Requirements:** Minimum 20 examples. Include examples for ALL classes defined in `classes_description`.

## config.yaml

```yaml
base:
  task: classification
```

## Unstructured Data (Optional)

Unlabelled examples or domain documentation to guide synthetic data generation. Single field: `context`.

```json
{"context": "Canceling my order is what I need to do right now."}
{"context": "I swear that there are 2 payments on the app that I didn't make. Could my card be stolen?"}
{"context": "Too many charges on my card, how do I go about fixing that?"}
{"context": "I got less cash than what I specified at the ATM."}
{"context": "Why is my last cheque deposit taking so long?"}
```

## Tips

1. **Balance your classes** -- Include similar numbers of examples per class.
2. **Clear class boundaries** -- Ensure classes do not overlap significantly.
3. **Descriptive class names** -- Use clear names and descriptions in `classes_description`.
4. **Representative examples** -- Cover the variety of inputs you expect in production.
5. **All classes represented** -- Every class in `classes_description` must have at least one example in the training data.
