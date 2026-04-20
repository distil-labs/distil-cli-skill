# Mutations Guide

Mutations control the structure of synthetically generated data. Each synthetic example is generated with a randomly sampled mutation modifier appended to the generation prompt. This steers the teacher model to produce examples that cover specific domains/patterns rather than repetitive structures.

## When to Use Mutations

Mutations are most valuable when your seed data does not cover all the scenarios you expect in production. You don't *need* to configure them on the first training run — the defaults are reasonable — but add them to the initial config whenever the situation calls for it. Revisit them during iteration if the trained model underperforms on specific slices.

Use them to:

- **Cover missing domains** -- If your seed data focuses on a few scenarios but production will see many more, add topic mutators to steer generation toward the missing areas.
- **Control data characteristics** -- If you need examples with specific properties (e.g., controlling the length of multi-turn conversations, or the complexity of questions), use a built-in mutator.
- **Improve model on specific scenarios** -- If your model underperforms on certain patterns, add topic mutators targeting those patterns in the next training iteration.

**Rule of thumb:** mutations are optional on the first run. If the task description explicitly names patterns, domains, or length characteristics to cover (e.g., "short/medium/long conversations", "billing vs. cancellation vs. tech support"), translating them into `mutation_topics` / `basic_mutators_to_use` upfront is usually worth the small amount of extra setup.

## Topic Mutators

Topic mutators are the primary way to control synthetic data diversity. They steer generation toward specific subjects relevant to your domain. Each generation call samples one topic from the list and adds it as a directive to the teacher.

**Use 1-2 topic mutator lists** with **3-10 topics each** for best results.

### Single topic list

All topics are sampled from one pool:

```yaml
synthgen:
  mutation_topics:
    - "billing disputes"
    - "account cancellation"
    - "technical support"
    - "onboarding questions"
```

### Multiple topic lists

Use nested lists when you want multiple independent topic dimensions. Each list is sampled independently, so the model gets one topic from each list per generation:

```yaml
synthgen:
  mutation_topics:
    - ["billing disputes", "account cancellation", "technical support"]
    - ["enterprise customers", "small business", "individual users"]
```

This produces combinations like "billing disputes" + "enterprise customers" or "technical support" + "individual users".

## Built-in Mutators

Three built-in mutators are available via the `basic_mutators_to_use` config field. **Use at most one built-in mutator at a time** — combining multiple built-in mutators creates conflicting instructions that can degrade data quality.

### `complexity`

Controls how difficult the generated examples are. Samples uniformly from:

| Level | Description |
|-------|-------------|
| trivial | Obvious answer, no reasoning needed |
| simple | Straightforward, minimal reasoning required |
| medium | Some reasoning, a few factors to consider |
| complex | Multiple factors, nuance, or ambiguity involved |
| highly complex | Expert-level, many interacting factors, edge cases |

### `length`

Controls the verbosity of generated examples. Samples from:

| Level | Description |
|-------|-------------|
| short and concise | 1-2 sentences, essentials only |
| medium length | 3-5 sentences, covers main points |
| detailed | Multiple paragraphs, includes context and nuance |

### `specificity`

Controls how domain-specific the generated language is. Samples from:

| Level | Description |
|-------|-------------|
| generic and vague | Avoid specific references and concrete terms |
| somewhat specific | Use descriptive but general references and terms |
| specific | Use realistic terms and references |
| very specific | Use precise, domain-heavy terms and references |

## How the Final Mutation Prompt Is Composed

The platform composes a mutation prompt from your configured mutators. Each active mutator samples one value, and they are combined into a single directive block appended to the generation prompt.

**Example:** With `basic_mutators_to_use: ["complexity"]` and a topic mutator configured:

```
Important! Make sure to follow those guidelines:
Generated examples should be complex: Multiple factors, nuance, or ambiguity involved.
Generated examples should focus on: billing disputes.
```

Each generation call samples fresh values, so across many calls the dataset naturally spans the configured variation space. For instance, the next call might produce:

```
Important! Make sure to follow those guidelines:
Generated examples should be trivial: Obvious answer, no reasoning needed.
Generated examples should focus on: technical support.
```

## Configuration

Both settings live under the `synthgen` section of `config.yaml`:

```yaml
synthgen:
  basic_mutators_to_use: ["complexity"]
  mutation_topics: ["billing disputes", "account cancellation", "technical support"]
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `basic_mutators_to_use` | `list[string]` | `["complexity"]` | Which built-in mutator to enable. Options: `complexity`, `length`, `specificity`. Use at most one. |
| `mutation_topics` | `list[list[string]]` or `list[string]` | `[]` | Topics to sample from during generation. Empty list disables topic mutations. |

## Recommendations

**Best practice:** Use 0-1 built-in mutators combined with 1-2 topic mutator lists for the best results.

**Start with the default** (`["complexity"]`) and review your synthetic data quality. If the data already follows your target distribution well, you may not need mutations at all — set `basic_mutators_to_use: []`.

**Topic mutators are the main lever** for improving data diversity. If your model underperforms on certain scenarios, add those scenarios as topics rather than adding more built-in mutators.

**Keep topic lists focused** — 3-10 topics per list is the sweet spot. Too many dilutes the signal; too few limits diversity.
