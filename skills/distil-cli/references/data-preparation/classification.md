# Data Preparation: Classification

Task value: `classification`. Rows are two-message conversations: user = the input text,
assistant = the class label.

```jsonl
{"messages": [{"role": "user", "content": "I have multiple charges for one transaction."}, {"role": "assistant", "content": "transaction_charged_twice"}]}
```

Rules beyond the shared checklist (`overview.md`):

- Labels must exactly match keys of `classes_description` in job_description.json
  (fields: `../job-description.md`): the label sets in train, test, and the job description
  must be identical.
- Each class needs at least `synthgen.num_positive_exemplars_per_generation` and
  `num_negative_exemplars_per_generation` train examples (defaults 2).
- Optional `unstructured.jsonl` (`context` rows) with unlabelled in-domain texts improves
  synthetic input realism.
- `synthgen.match_generated_distribution_to_seed: true` keeps generated class balance equal
  to the seed's.
