# Tool Calling Data Preparation

Use tool calling when the model should select and invoke functions/APIs based on user requests.

**Example use cases:**
- Voice assistants controlling smart home APIs
- Chatbot action dispatching to CRM/database operations
- Workflow automation and microservice routing
- Natural language to API translation

**Important:** Only Llama3 family student models are supported for tool calling.

## Required Files

### 1. job_description.json

```json
{
  "task_description": "Based on user requests, select the appropriate function and provide the correct parameters to fulfill the request.",
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get the current weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "City name, e.g., 'San Francisco, CA'"
            },
            "unit": {
              "type": "string",
              "enum": ["celsius", "fahrenheit"],
              "description": "Temperature unit"
            }
          },
          "required": ["location"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "send_email",
        "description": "Send an email to a recipient",
        "parameters": {
          "type": "object",
          "properties": {
            "to": {
              "type": "string",
              "description": "Recipient email address"
            },
            "subject": {
              "type": "string",
              "description": "Email subject line"
            },
            "body": {
              "type": "string",
              "description": "Email body content"
            }
          },
          "required": ["to", "subject", "body"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "create_calendar_event",
        "description": "Create a new calendar event",
        "parameters": {
          "type": "object",
          "properties": {
            "title": {
              "type": "string",
              "description": "Event title"
            },
            "date": {
              "type": "string",
              "description": "Event date in YYYY-MM-DD format"
            },
            "time": {
              "type": "string",
              "description": "Event time in HH:MM format"
            },
            "duration_minutes": {
              "type": "integer",
              "description": "Event duration in minutes"
            }
          },
          "required": ["title", "date", "time"]
        }
      }
    }
  ]
}
```

**Fields:**
- `task_description`: Explain what the model should do
- `tools`: Array of tool definitions in OpenAI function calling format

### 2. train.csv

```csv
question,answer
"What's the weather like in New York?","{""name"": ""get_weather"", ""parameters"": {""location"": ""New York, NY"", ""unit"": ""fahrenheit""}}"
"Send an email to john@example.com saying the meeting is confirmed","{""name"": ""send_email"", ""parameters"": {""to"": ""john@example.com"", ""subject"": ""Meeting Confirmation"", ""body"": ""The meeting is confirmed.""}}"
"Schedule a team standup for tomorrow at 9am for 30 minutes","{""name"": ""create_calendar_event"", ""parameters"": {""title"": ""Team Standup"", ""date"": ""2024-01-15"", ""time"": ""09:00"", ""duration_minutes"": 30}}"
```

**Requirements:**
- Minimum 20 examples
- `question`: The user's natural language request
- `answer`: JSON string with `name` (function name) and `parameters` (arguments)
- Include examples for all tools

### 3. test.csv

Same format as train.csv:

```csv
question,answer
"Check the temperature in London in celsius","{""name"": ""get_weather"", ""parameters"": {""location"": ""London, UK"", ""unit"": ""celsius""}}"
```

### 4. config.yaml

```yaml
task: tool-calling-closed-book

# Model selection - MUST use Llama3 family
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
  output_is_json: true
```

**Key settings for tool calling:**
- `task`: Must be `tool-calling-closed-book`
- `student_model_name`: Must be a Llama3 family model
- `output_is_json`: Set to `true` to enforce JSON output format

### 5. unstructured.csv (Optional)

Example scenarios for synthetic data generation:

```csv
context
"User wants to check weather before their trip to Paris next week."
"User needs to send a follow-up email to a client about the proposal."
"User is scheduling a recurring weekly meeting with the engineering team."
```

## Complete Example Directory

```
my-tool-calling-data/
├── job_description.json
├── train.csv
├── test.csv
├── config.yaml
└── unstructured.csv (optional)
```

## Upload and Train

```bash
# Upload data
distil model upload-data <model-id> --data ./my-tool-calling-data

# Run teacher evaluation
distil model run-teacher-evaluation <model-id>
distil model teacher-evaluation <model-id>

# Train model
distil model run-training <model-id>
```

## Using the Trained Model

The model outputs JSON tool calls:

```bash
python model_client.py --question "What's the weather in Tokyo?"
# Output: {"name": "get_weather", "parameters": {"location": "Tokyo, Japan"}}
```

Parse the output and execute the function:
```python
import json

response = model.generate("What's the weather in Tokyo?")
tool_call = json.loads(response)

if tool_call["name"] == "get_weather":
    result = get_weather(**tool_call["parameters"])
```

## Tips

1. **Clear tool descriptions** - Make function descriptions unambiguous
2. **Comprehensive parameter descriptions** - Help the model understand what each parameter expects
3. **Varied examples** - Show different ways users might request the same action
4. **Edge cases** - Include examples with optional parameters both present and absent
5. **Llama3 only** - Remember only Llama3 family models are supported for this task type
6. **Valid JSON** - Ensure all answer fields contain valid JSON strings
