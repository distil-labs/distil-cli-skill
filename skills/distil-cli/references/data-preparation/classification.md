# Data Preparation: Classification

Task value: `classification`. Rows are two-message conversations: user = the input text,
assistant = the class label.

```jsonl
{"messages": [{"role": "user", "content": "I have multiple charges for one transaction."}, {"role": "assistant", "content": "transaction_charged_twice"}]}
```

Rules beyond the shared checklist (`overview.md`):

- Labels must exactly match keys of `classes_description` in job_description.json
  (fields: `../job-description.md`). The label sets in train, test, and the job description
  must be identical.
- A populated split must hold every class (`overview.md` § Validation rules), and a class
  with only one or two train rows gives generation almost nothing to imitate for it.
- Traces left in the Dataset serve as context for generation and improve synthetic input
  realism (`../platform.md` § The expand operations).
- `synthgen.match_generated_distribution_to_seed: true` keeps generated class balance equal
  to the train split's.
