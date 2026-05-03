# Tools

Validation and authoring helpers. Out of scope for the v0 spec commit —
placeholder docs for what will land here as the ecosystem grows.

Planned:

- `validate-protocol` — lint a protocol YAML against
  [`../schema/protocol.schema.json`](../schema/protocol.schema.json) and
  cross-check that every `inputs[*].metric` exists in
  [`../schema/vocabulary.yaml`](../schema/vocabulary.yaml).
- `validate-sources` — lint a sources config against
  [`../schema/sources.schema.json`](../schema/sources.schema.json), check
  that every referenced env var is set, and ping each declared
  endpoint for reachability.
- `evaluate-protocol` — given a protocol YAML, a sources YAML, and a
  workflow name, run the workflow once against the configured adapters
  and print the result. The smallest-possible end-to-end runtime;
  useful for offline iteration before plugging into a skill or app.

Reference implementations live in language-specific repos
(`cogignition/hegemonikron` for Swift; community contributions
welcome). This directory holds spec-level test fixtures and CLI
contracts only.
