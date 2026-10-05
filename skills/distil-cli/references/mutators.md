# Mutators

Mutators control the diversity of synthetic data: each generation call samples one value per
configured mutator and adds it as a directive to the generation prompt. They are configured in
`synthgen.mutators` in `config.yaml`. None are applied by default: without mutators every
example comes from the same prompt, and the only variation is the teacher temperature and the
sampled in-context exemplars.

## The shape

`synthgen.mutators` is a list, one entry per dimension to vary:

```yaml
synthgen:
  mutators:
    - name: complexity
      values: [simple, medium, complex]
    - name: topic
      values:
        - "about billing disputes: refunds, double charges, invoice errors"
        - "about account cancellation: downgrades and win-back attempts"
        - "about technical support: setup problems and integration errors"
```

| Field | Required | Notes |
|---|---|---|
| `name` | yes | The dimension. Unique across mutators: it labels the sampled value and seeds the mutator's own random stream |
| `values` | no | The values, each a plain string. Leave the field out to have the teacher detect them, which requires a `description` (§ Auto-detected values). An empty list is a config error |
| `method` | no | `fixed` (default) samples from the target distribution. `adaptive` measures what survived validation and samples more of the values that fall behind (§ Adaptive mutators). Any other value is rejected |
| `target_distribution` | no | `uniform` (default), `match_seed` (§ Matching the seed distribution), or one weight per listed value, e.g. `[5, 3, 2]` (normalised) |
| `description` | no | What the dimension means. Required when `values` is left out. Otherwise read only by the classifier, so it matters with `adaptive` and `match_seed` |
| `auto_detect_sample_size` | no | Seed examples the detector sees when `values` is left out. Default `50` |
| `beta` | no | `adaptive` only, 0 to 1, default `0.9`. How far sampling may move away from the target (§ Adaptive mutators) |

Unknown fields are rejected. A weight list needs exactly one weight per value.

A value is one string. Put the detail of a value into the same string after a colon, and quote
it so YAML reads a string: `- "simple: minimal reasoning required"` is a value, while
`- simple: minimal reasoning required` is a YAML mapping and is rejected.

## Prompt composition

Active mutators render one line each under a shared prelude:

```
Important! Make sure to follow those guidelines:
Generated examples should be simple
Generated examples should be about billing disputes: refunds, double charges, invoice errors
```

Every line is `Generated examples should be ` followed by the value verbatim, so phrase values
to finish that sentence (`about billing disputes`, not `billing`).

Each mutator draws from its own random stream, derived from `base.random_seed` and the mutator
name, so a config reproduces its own draws and renaming a mutator changes them.

## How many, and in what proportions

- One mutator per independent dimension. Two mutators of three values each cover a 3x3 grid
  without enumerating nine combinations.
- 1-2 mutators of 3-10 values each. More gives each value fewer generation calls;
  fewer limits diversity.
- To skew the mix, give one weight per value: `values: [English, French]` with
  `target_distribution: [3, 1]` asks for English in about three calls out of four.
- A single-value mutator applies the same directive to every call: a constant extra generation
  instruction (for example output length) that leaves the job description unchanged.

## Auto-detected values

When the dimension is known but its values are not, leave `values` out and describe the
dimension. The teacher proposes the values once, at the start of the run:

```yaml
synthgen:
  mutators:
    - name: topic
      description: which part of the product the customer is asking about
```

- The detector reads the job description and a random sample of up to `auto_detect_sample_size`
  seed examples, and works with an empty seed set too. It aims at what the task meets in
  production, so expect values the seed data lacks, typically 3-8.
- One teacher call per detecting mutator. An unparsable or empty answer fails the run.
- The detected values are logged (`Detected <name> values from the job description and N seed
  examples: [...]`) and not written back into the config.
- A weight list is a config error here, since there are no listed values to weight.

Read the detected values in the smoke's log, then copy them into `values` to edit or fix them.
Listed values need no teacher call and give the same values on every run.

## Adaptive mutators

A fixed mutator never checks what came back. The teacher can ignore a directive and validation
drops examples unevenly, so the data can end up short on the values the mutator was added for.
An adaptive mutator measures that and corrects the sampling. Use it when a fixed mutator left
the generated data unbalanced.

```yaml
synthgen:
  mutator_update_frequency: 5
  mutators:
    - name: topic
      method: adaptive
      description: which part of the product the customer is asking about
      target_distribution: [0.5, 0.3, 0.2]
      values:
        - "about billing disputes: refunds, double charges, invoice errors"
        - "about account cancellation: downgrades and win-back attempts"
        - "about technical support: setup problems and integration errors"
```

- Every `synthgen.mutator_update_frequency` batches, the teacher labels each surviving example
  with the value it matches, or "other". Counts for the batches in between are extrapolated.
  Each labelled example is one teacher call, so a higher frequency costs less and measures less
  often.
- Sampling moves towards the values furthest behind the target. The weights are
  `(1 - beta) * target + beta * correction`, so with the default 0.9 every value keeps at least
  a tenth of its target share. Lower `beta` when one value is rejected again and again and the
  mix matters more than reaching it.
- Always write the `description`: the classifier decides from the values and the description
  alone.
- The classifier sees validated examples only, so a value that validation keeps rejecting
  (format, length) is asked for again and again. Fix the value instead; adapting only spends
  teacher calls on it.
- A smoke (128 examples, one batch at the default `generation_iteration_size`) ends before the
  first classifier run. Set `generation_iteration_size: 32` and `mutator_update_frequency: 1` in
  the smoke config to see it adapt, and restore both for the full run.

## Matching the seed distribution

`target_distribution: match_seed` has the teacher classify the seed examples along the dimension
once, at the start of the run, and uses the proportions it finds as the target. It works with
either `method` and costs one teacher call per seed example.

- Values absent from the seed data get zero weight and are never asked for. To get a value the
  seed lacks, give explicit weights.
- With no seed data, or none that matches any value, the mutator samples uniformly.
- It works per mutator, on any task and any dimension. `match_generated_distribution_to_seed`
  is the classification-only setting for the class mix.

## Mutators for classification

In classification, a mutator whose values bear on the label (a severity, a station, a
category) can lead the teacher to the wrong class. Prefer mutators that leave the label to
`classes_description` (style, length, phrasing), and check the labels of the generated rows in
the smoke whenever mutators are on.

## Tool mutator for chat completion

For chat completion with tools, add a mutator with one value per tool:
`data-preparation/chat-completion.md` § Synthetic data with tools.

## Recipes

Three general-purpose mutators to copy. Use one of them at a time: their directives conflict
when combined.

```yaml
synthgen:
  mutators:
    - name: complexity
      values:
        - "simple: straightforward, minimal reasoning required"
        - "medium: some reasoning, a few factors to consider"
        - "complex: multiple factors, nuance, or ambiguity involved"
    - name: length            # or this one, not both
      values:
        - "short: 1-2 sentences, essentials only"
        - "medium: 3-5 sentences, covers main points"
        - "detailed: multiple paragraphs, includes context and nuance"
    - name: specificity       # or this one
      values:
        - "generic: avoid specific references and concrete terms"
        - "specific: use realistic terms and references"
        - "very specific: use precise, domain-heavy terms and references"
```

## Which option for which problem

- Values unknown: § Auto-detected values. A value comes up short: `method: adaptive` or a
  heavier weight. Proportions should follow the seed data: `target_distribution: match_seed`.
- Wrong length or difficulty profile: a § Recipes mutator. Near-duplicates of the seed data:
  `synthgen.validation_similarity_threshold` (`configuration.md`), not a mutator.
