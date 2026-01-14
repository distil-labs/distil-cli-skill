# Classification Task Data Preparation

Use classification when you need to assign text to categories from a fixed set.

**Example use cases:**
- Intent detection for customer service
- Content moderation (toxic/safe, spam/not spam)
- Sentiment analysis (positive/negative/neutral)
- Support ticket triage by department

## Required Files

### 1. job_description.json

```json
{
  "task_description": "Classify customer support messages into intent categories to route them to the appropriate team.",
  "classes_description": {
    "billing": "Questions about invoices, payments, charges, or refunds",
    "technical": "Issues with product functionality, bugs, or errors",
    "account": "Account access, password resets, profile changes",
    "sales": "Pricing inquiries, upgrades, new purchases",
    "general": "General questions that don't fit other categories"
  }
}
```

**Fields:**
- `task_description`: Explain what the model should do
- `classes_description`: Map each class name to a clear description

### 2. train.csv

```csv
question,answer
"Why was I charged twice this month?",billing
"The app crashes when I try to upload a file",technical
"How do I change my email address?",account
"What's the difference between Pro and Enterprise?",sales
"What are your business hours?",general
```

**Requirements:**
- Minimum 20 examples
- `question`: The input text to classify
- `answer`: The class label (must match keys in `classes_description`)
- Include examples for all classes

### 3. test.csv

Same format as train.csv. Used for evaluation, not training.

```csv
question,answer
"I need a refund for my last payment",billing
"Button doesn't respond when clicked",technical
```

### 4. config.yaml

```yaml
task: classification

# Model selection
student_model_name: Llama-3.2-1B-Instruct
teacher_model_name: Llama-3.3-70B-Instruct

# Training parameters
tuning:
  learning_rate: 5e-5
  num_train_epochs: 4
  use_lora: true
  lora_r: 64

# Synthetic data generation
synthetic_generation:
  generation_target: 10000
  teacher_temperature: 0.7
  match_generated_distribution_to_seed: true
```

**Key settings for classification:**
- `task`: Must be `classification`
- `match_generated_distribution_to_seed`: Set to `true` to maintain class balance

### 5. unstructured.csv (Optional)

Domain-specific text to improve synthetic data generation:

```csv
context
"Our billing cycle runs from the 1st to the 30th of each month. Charges appear within 3 business days."
"To reset your password, click the 'Forgot Password' link on the login page."
"Enterprise plans include priority support and custom integrations."
```

## Complete Example Directory

```
my-classification-data/
├── job_description.json
├── train.csv
├── test.csv
├── config.yaml
└── unstructured.csv (optional)
```

## Upload and Train

```bash
# Upload data
distil model upload-data <model-id> --data ./my-classification-data

# Run teacher evaluation
distil model run-teacher-evaluation <model-id>
distil model teacher-evaluation <model-id>

# Train model
distil model run-training <model-id>
```

## Tips

1. **Balance your classes** - Include similar numbers of examples per class
2. **Clear class boundaries** - Ensure classes don't overlap significantly
3. **Representative examples** - Cover the variety of inputs you expect in production
4. **Descriptive class names** - Use clear names in `classes_description`
