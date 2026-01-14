# Question Answering (Closed-Book) Data Preparation

Use closed-book QA when the model should learn facts from your data and answer questions using its internal knowledge—no context provided at inference time.

**Example use cases:**
- FAQ bots that "know" your product
- Domain-specific knowledge assistants
- Customer support without retrieval
- Fact recall from training data

## Required Files

### 1. job_description.json

```json
{
  "task_description": "Answer questions about our company's products, policies, and services. Use your knowledge to provide accurate, helpful responses.",
  "llm_as_a_judge_instructions": "Evaluate if the answer is factually correct based on the company's actual policies and product information. The answer should be accurate and complete."
}
```

**Fields:**
- `task_description`: Explain what the model should do
- `llm_as_a_judge_instructions` (optional): Guidance for evaluation

### 2. train.csv

```csv
question,answer
"What are your business hours?","Our support team is available Monday through Friday, 9 AM to 6 PM EST."
"Do you offer a free trial?","Yes, we offer a 14-day free trial with full access to all features. No credit card required."
"What is your refund policy?","We offer full refunds within 30 days of purchase for annual plans."
```

**Requirements:**
- Minimum 20 examples
- `question`: The user's question
- `answer`: The factual response the model should learn

### 3. test.csv

Same format as train.csv:

```csv
question,answer
"How do I contact support?","You can contact support via email at support@example.com or through the in-app chat."
```

### 4. config.yaml

```yaml
task: question-answering-closed-book

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
  generation_per_unstructured_context: 5
```

**Key settings for closed-book:**
- `task`: Must be `question-answering-closed-book`
- `generation_per_unstructured_context`: Number of examples to generate per unstructured context (important for this task type)

### 5. unstructured.csv (Highly Recommended)

This file is crucial for closed-book QA. It contains the knowledge you want the model to learn:

```csv
context
"Our company was founded in 2019 and is headquartered in San Francisco. We have over 500 employees globally."
"Pricing: Starter plan is $10/month, Pro plan is $25/month, Enterprise plan is custom pricing. All plans include basic support."
"Security: We are SOC 2 Type II certified. Data is encrypted at rest and in transit. We perform annual security audits."
"Integration: We support Slack, Microsoft Teams, Salesforce, and Zendesk integrations. API access is available on Pro and Enterprise plans."
```

The model will generate synthetic QA pairs from these contexts and learn the facts.

## Complete Example Directory

```
my-closed-book-data/
├── job_description.json
├── train.csv
├── test.csv
├── config.yaml
└── unstructured.csv  # Highly recommended for this task type
```

## Upload and Train

```bash
# Upload data
distil model upload-data <model-id> --data ./my-closed-book-data

# Run teacher evaluation
distil model run-teacher-evaluation <model-id>
distil model teacher-evaluation <model-id>

# Train model
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

1. **Comprehensive unstructured data** - Include all the facts you want the model to know in `unstructured.csv`
2. **Consistent information** - Avoid contradictions in your training data
3. **Cover your domain** - The model can only answer questions about topics in its training data
4. **Factual accuracy** - Double-check facts in your data—the model will learn exactly what you provide
5. **Consider scope** - Closed-book models work best for bounded domains with clear factual answers
