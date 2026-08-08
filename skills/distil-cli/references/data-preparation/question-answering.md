# Data Preparation: Question Answering

Task value: `question-answering`. Rows are two-message conversations: user = the input,
assistant = the expected output. The catch-all task for text-in text-out problems
(extraction, transformation, generation).

```jsonl
{"messages": [{"role": "user", "content": "What is the capital of France?"}, {"role": "assistant", "content": "Paris"}]}
```

Rules beyond the shared checklist (`overview.md`):

- Synthgen generates question-answer pairs one-shot from `task_description` and the seed
  examples. Add `synthetic_data_generation_instructions` to job_description.json (fields:
  `../job-description.md`) to describe the generated inputs: formats, domains, variation,
  noise.
- For JSON outputs set `synthgen.output_is_json: true`. Every assistant answer must then
  parse as JSON.
- Optional `unstructured.jsonl` supplies in-domain texts sampled as inspiration.
