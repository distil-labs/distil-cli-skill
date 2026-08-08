# Mutators

Mutators control the structure and diversity of synthetic data: each generation call samples
one value per active mutator and appends the directives to the generation prompt. Configured
in the `synthgen` section of `config.yaml`.

## Custom (topic) mutators: the main lever

`synthgen.mutation_topics` steers generation toward named subjects. Use it when seed data
misses scenarios production will see, or the model underperforms on specific slices.

```yaml
# Single pool: each call samples one topic
synthgen:
  basic_mutators_to_use: []   # turn off the built-ins when introducing custom mutators
  mutation_topics: ["billing disputes", "account cancellation", "technical support"]

# Nested lists: independent dimensions, one sample from EACH list per call
synthgen:
  basic_mutators_to_use: []
  mutation_topics:
    - ["billing disputes", "account cancellation", "technical support"]
    - ["enterprise customers", "small business", "individual users"]
```

Each list renders as `Generated examples should focus on: <topic>`. Nested lists produce
combinations ("billing disputes" + "enterprise customers"), covering a grid of scenarios
without enumerating it. A flat list is treated as one pool.

Use 1-2 lists of 3-10 topics each: more dilutes the signal per topic, fewer limits diversity.

A single-item list is a useful special case: it applies the same directive to EVERY call,
turning the mutator into a constant extra generation instruction (e.g. steering output
length or document realism) without touching the job description.
And when you introduce custom mutators, set `basic_mutators_to_use: []` explicitly: the
default `["complexity"]` stays active otherwise, and the stacked directives start tripping
over one another.

## Built-in mutators

`synthgen.basic_mutators_to_use` (default `["complexity"]`). Use at most one; combined
built-ins give conflicting instructions. `[]` disables them.

| Mutator | Samples uniformly from |
|---|---|
| `complexity` | trivial, simple, medium, complex, highly complex |
| `length` | short and concise (1-2 sentences), medium length (3-5 sentences), detailed (multiple paragraphs) |
| `specificity` | generic and vague, somewhat specific, specific, very specific |

## Prompt composition

Active mutators render one line each under a shared prelude:

```
Important! Make sure to follow those guidelines:
Generated examples should be complex: Multiple factors, nuance, or ambiguity involved.
Generated examples should focus on: billing disputes.
```

Sampling is seeded from `base.random_seed`, so a config reproduces its mutation sequence.

## When to touch this

- First run: defaults are fine unless the task already names patterns, domains, or length
  characteristics to cover; then set topics upfront.
- Diversity gaps while iterating: add or refine topic lists.
- Wrong length or difficulty profile: swap the built-in.
- Near-duplicates of seed data: that is `validation_similarity_threshold`
  (`configuration.md`), not a mutator.
