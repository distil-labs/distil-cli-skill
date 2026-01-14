# Question Answering (RAG/Open-Book) Data Preparation

Use open-book QA when the model should answer questions using provided context passages. This is the RAG (Retrieval-Augmented Generation) pattern.

**Example use cases:**
- Customer support from product documentation
- Legal document analysis
- Technical documentation assistants
- Knowledge base automation

## Required Files

### 1. job_description.json

```json
{
  "task_description": "Answer customer questions about our product based on the provided documentation excerpt. Provide accurate, helpful responses grounded in the given context.",
  "llm_as_a_judge_instructions": "Evaluate if the answer correctly addresses the question using information from the context. The answer should be accurate, complete, and not include information outside the provided context."
}
```

**Fields:**
- `task_description`: Explain what the model should do
- `llm_as_a_judge_instructions` (optional): Guidance for evaluation

### 2. train.csv

```csv
question,context,answer
"How do I reset my password?","To reset your password: 1) Click 'Forgot Password' on the login page. 2) Enter your email address. 3) Check your inbox for a reset link. 4) Click the link and create a new password.","To reset your password, click 'Forgot Password' on the login page, enter your email, and follow the reset link sent to your inbox to create a new password."
"What payment methods do you accept?","We accept all major credit cards (Visa, MasterCard, American Express), PayPal, and bank transfers for annual plans. Cryptocurrency payments are not supported.","We accept Visa, MasterCard, American Express, PayPal, and bank transfers for annual plans."
```

**Requirements:**
- Minimum 20 examples
- `question`: The user's question
- `context`: The relevant passage/document excerpt
- `answer`: The response grounded in the context

### 3. test.csv

Same format as train.csv:

```csv
question,context,answer
"What is the refund policy?","Refunds are available within 30 days of purchase for annual plans. Monthly plans can be cancelled anytime but are not refundable.","Refunds are available within 30 days for annual plans. Monthly plans can be cancelled but are not refundable."
```

### 4. config.yaml

```yaml
task: question-answering-open-book

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
  num_distractor_context_blocks: 0
```

**Key settings for RAG:**
- `task`: Must be `question-answering-open-book`
- `num_distractor_context_blocks`: Set > 0 for RAFT training (adds irrelevant context to improve robustness)

### 5. unstructured.csv (Optional)

Additional context passages for synthetic data generation:

```csv
context
"Our enterprise plan includes SSO integration, custom branding, and dedicated support."
"API rate limits are 1000 requests per minute for Pro plans and 5000 for Enterprise."
"Data is encrypted at rest using AES-256 and in transit using TLS 1.3."
```

## Complete Example Directory

```
my-rag-data/
├── job_description.json
├── train.csv
├── test.csv
├── config.yaml
└── unstructured.csv (optional)
```

## Upload and Train

```bash
# Upload data
distil model upload-data <model-id> --data ./my-rag-data

# Run teacher evaluation
distil model run-teacher-evaluation <model-id>
distil model teacher-evaluation <model-id>

# Train model
distil model run-training <model-id>
```

## Using the Trained Model

For RAG tasks, you must provide context when querying:

```bash
python model_client.py --question "How do I upgrade?" --context "Upgrades can be done from Settings > Billing > Change Plan."
```

Or via API:
```python
messages = [
    {"role": "user", "content": "Context: Upgrades can be done from Settings > Billing.\n\nQuestion: How do I upgrade?"}
]
```

## Tips

1. **Ground answers in context** - Answers should only use information from the provided context
2. **Realistic context lengths** - Use context sizes similar to what you'll retrieve in production
3. **Varied question types** - Include factual, procedural, and explanatory questions
4. **Consider RAFT training** - Set `num_distractor_context_blocks` > 0 to train the model to ignore irrelevant context
