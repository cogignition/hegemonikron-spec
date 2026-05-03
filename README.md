# Hegemonikron Spec

The contract for the Hegemonikron protocol ecosystem: a vocabulary of
canonical health/training metrics, a schema for declaring coaching
protocols against that vocabulary, and a contract that data-source
adapters honor so protocols stay portable across stacks.

## What this is for

A coaching protocol — a set of rules, thresholds, and templates — should
be a *publishable, forkable artifact*, not configuration buried inside
one app. This repo defines the language those artifacts speak.

If you have:

- a protocol idea you want to publish so others can run it
- an app that wants to consume protocols authored by anyone
- a data source (HealthKit, InfluxDB, Garmin, CSV) you want to make
  available to existing protocols

… this repo defines the contract.

## Three layers

```
┌─ Vocabulary ───────────────────────────────────────────────┐
│ schema/vocabulary.yaml                                     │
│ Canonical metric names + units + supported aggregations.   │
│ Stable. Evolves slowly via PR. Protocols speak this.       │
└────────────────────────┬───────────────────────────────────┘
                         │
┌─ Protocols ─────────────┴──────────────────────────────────┐
│ schema/protocol.schema.json                                │
│ YAML files declaring inputs (against the vocabulary),      │
│ workflows, rules, and output templates. Authored by        │
│ humans (often co-authored with frontier models). Run by    │
│ apps and skills. Catalog: cogignition/protocols.           │
└────────────────────────┬───────────────────────────────────┘
                         │
┌─ Adapters ──────────────┴──────────────────────────────────┐
│ adapters/*.md                                              │
│ Each adapter implements: "given a metric request, return   │
│ the value(s) in canonical units." Users configure which    │
│ adapter fulfills which metric in their environment.        │
│ schema/sources.schema.json validates that user config.     │
└────────────────────────────────────────────────────────────┘
```

The decoupling: a protocol authored against the vocabulary runs on any
stack that has adapters covering its inputs. A user running on InfluxDB
and someone running on HealthKit consume the same protocol bytes.

## Authoring a protocol

A protocol is a YAML file. Minimum shape:

```yaml
apiVersion: hegemonikron.cogignition.cloud/v1
kind: Protocol
metadata:
  id: 2026-05-03-zone2-fortnight
  title: "Zone 2 Base with Fortnight HRV Gate"
  author: cogignition
  license: CC-BY-4.0

inputs:
  hrv_today:    { metric: hrv,         aggregation: latest }
  hrv_baseline: { metric: hrv,         aggregation: mean,    window: 14d }
  sleep_hours:  { metric: sleep_total, aggregation: latest }

workflows:
  morning_readiness:
    cadence: daily@07:30
    inputs: [hrv_today, hrv_baseline, sleep_hours]
    rules:
      - if: "hrv_today / hrv_baseline < 0.85"
        then: { intensity_cap: zone_2_only, reason: "HRV below 85% of baseline" }
      - if: "sleep_hours < 6"
        then: { intensity_cap: cap_at_zone_3, reason: "sleep deficit" }
      - else: { intensity_cap: full_protocol }
    output:
      style: "morning brief, 3 sentences, no filler"
      mention: [readiness_score, intensity_cap, dominant_signal]
```

The full schema is in [`schema/protocol.schema.json`](schema/protocol.schema.json);
a fully-worked example is in [`examples/protocol-zone2-fortnight.yaml`](examples/protocol-zone2-fortnight.yaml).

## Configuring sources

A user's runtime maps canonical metrics to actual data. Minimum shape:

```yaml
# ~/.config/hegemonikron/sources.yaml
default_adapter: healthkit
```

That's enough for an Apple-Watch-only setup. Mixed setups override
specific metrics:

```yaml
default_adapter: healthkit
adapters:
  influxdb:
    endpoint: ${HEGEMONIKRON_INFLUX_URL}
    bucket: health
    org: kenning
    auth_env: INFLUXDB_TOKEN
    metrics: [carbohydrates, protein, total_fat, dietary_energy]   # nutrition only
```

Three example configurations are in [`examples/`](examples/). Adapters
are documented in [`adapters/`](adapters/).

## The ecosystem

| Repo | What |
|---|---|
| **`cogignition/hegemonikron-spec`** (this) | the contract |
| `cogignition/protocols` | public catalog of YAML protocols |
| `cogignition/hegemonikron-skills` | installable Claude Code / Cowork skills |
| `cogignition/hegemonikron` | reference macOS + iOS app |

Anyone implementing this spec can publish protocols, build apps, or run
skills. The spec is the only thing that has to be agreed.

## Versioning

`apiVersion: hegemonikron.cogignition.cloud/v1` is the current contract.
Breaking changes bump to `v2`; the catalog and runtimes will accept
multiple major versions side-by-side. Additions (new metrics, new
aggregations, new optional fields) are non-breaking and don't bump.

## Contributing

PRs welcome:

- New canonical metrics (with rationale: who needs it, who can supply it)
- New adapter docs (one markdown file per adapter)
- Schema clarifications, examples, validation tooling

Substantive changes to the schema itself open as RFC discussions before PR.

## License

CC-BY-4.0. See [`LICENSE`](LICENSE).
