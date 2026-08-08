# Data Preparation: Open Book QA (RAG)

Task value: `question-answering-open-book`. RAG applications only (see `../task-types.md`);
for anything without a retriever use `question-answering`. Rows are two-message conversations
plus a top-level `context` field. Requires `unstructured.jsonl`.

```jsonl
{"context": "The end of the Dark Ages ... first Olympic Games ...", "messages": [{"role": "user", "content": "When did the Olympic Games begin?"}, {"role": "assistant", "content": "776 BC"}]}
```

Rules beyond the shared checklist (`overview.md`); job_description.json uses the QA fields
(`../job-description.md`):

- Every train/test row needs a `context` the answer is grounded in; `context` should be a
  real chunk from the RAG database.
- `unstructured.jsonl` (`context` rows) must be individual entries from the actual RAG
  database (the same chunks the retriever serves in production), not random documents;
  synthgen builds new QA pairs from these entries, so off-distribution documents produce
  off-distribution training data.
- To harden against distractor passages (RAFT), set
  `synthgen.num_distractor_context_blocks` above 0.
- At inference the context is inlined into the first user message as
  `<context>...</context>` (see `../deployment.md`).
