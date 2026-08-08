# Data Preparation: Closed Book QA

Task value: `question-answering-closed-book`. For memorizing a knowledge source: the RAG-like
setting where a database exists but nothing is retrieved at inference, so the model answers
from memory (see `../task-types.md`). Rows are two-message conversations (no context field).
Requires `unstructured.jsonl`, and that content is the knowledge distilled into the model.

```jsonl
{"messages": [{"role": "user", "content": "When did the siege of Sevastopol end?"}, {"role": "assistant", "content": "September 1855"}]}
```

Rules beyond the shared checklist (`overview.md`). The job description uses the QA fields
(`../job-description.md`):

- `unstructured.jsonl` (`context` rows) is the knowledge base, and coverage here bounds what
  the model can learn. Chunk it so each row is a self-contained fact-bearing passage.
- Generation volume scales with the knowledge base:
  `synthgen.generation_per_unstructured_context` (when set) overrides `generation_target`
  with `per_context x len(unstructured)`.
- Train and test QA pairs must be answerable from the unstructured content.
