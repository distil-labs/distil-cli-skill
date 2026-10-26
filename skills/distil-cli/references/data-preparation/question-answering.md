# Data Preparation: Question Answering

Task value: `question-answering`. Rows are two-message conversations: user = the input,
assistant = the expected output. The catch-all task for text-in text-out problems
(extraction, transformation, generation).

```jsonl
{"messages": [{"role": "user", "content": "What is the capital of France?"}, {"role": "assistant", "content": "Paris"}]}
```

Rules beyond the shared checklist (`overview.md`):

- Synthgen generates question-answer pairs one-shot from `task_description` and the rows in
  the train split. Add `synthetic_data_generation_instructions` to job_description.json (fields:
  `../job-description.md`) to describe the generated inputs: formats, domains, variation,
  noise.
- For JSON outputs set `synthgen.output_is_json: true`. Every assistant answer must then
  parse as JSON.
- Traces left in the Dataset are sampled as inspiration for generation
  (`../platform.md` § The expand operations).

## Classifying with question-answering

There is no classification task. A config with `task: classification` is rejected with `The task
classification is no longer supported. Use the task question-answering instead, with a mutator
over the class labels`, and so is a job on an existing classification Dataset, before anything
is charged: upload its rows into a new question-answering Dataset. A label from a fixed set is a
`question-answering` task:

- **Rows**: user = the input text, assistant = the label and nothing else.
- **`task_description`**: the labels, a definition of each, and "Answer with the label only."
  Question answering adds no class list to the prompt and has no `classes_description` field, so
  `task_description` is the only place the teacher and the student read the labels from.
- **One mutator over all the labels**, with one value per label. Each value completes the
  sentence "Generated examples should be {value}" (`../mutators.md` § Prompt composition):

  ```yaml
  synthgen:
    mutators:
      - name: label
        description: which label the message has
        values:
          - "a message whose correct label is billing"
          - "a message whose correct label is technical_issue"
  ```

  Generation is then spread roughly evenly over the labels. `method: adaptive` evens it
  further, at the cost of a teacher call per surviving row it classifies (`../mutators.md` § Adaptive mutators). This mutator is the
  normal exception to the advice of 1-2 mutators of 3-10 values (`../mutators.md` § How many,
  and in what proportions).
- **The metric**: `binary` exact match is the accuracy (`../evaluation-metrics.md` § Primary
  metric per task).
- **The test set**: nothing checks that every class is present, so a test set the user
  supplies may miss rare classes. Count the labels in `test.jsonl` and tell the user which are
  missing.

What the old task did and this does not:

- **No per-class report** in the metrics. Count per-class errors from the predictions file.
- **No check that an answer is a label.** Count the labels of the generated rows in the smoke,
  and treat an answer outside the label set as a defect of the run
  (`../../stages/synthetic-data-generation.md` Step 4).
