# Adapter: mock

Returns deterministic synthetic data. Used by:

- protocol authors who want to verify a YAML lints and evaluates
  cleanly before pointing it at real data
- runtime developers writing tests
- CI in the `cogignition/protocols` catalog (every protocol is
  evaluated against a mock fixture before merge)

## Configuration

```yaml
adapters:
  mock:
    config:
      seed: 42
      profile: healthy_baseline
```

| Field | Required | Description |
|---|---|---|
| `config.seed` |   | Integer seed for deterministic RNG. Default: `0`. |
| `config.profile` |   | Named scenario. See *Profiles* below. Default: `healthy_baseline`. |
| `config.fixture` |   | Path to a YAML fixture file overriding profile output. |

## Profiles

Built-in profiles produce plausible-but-synthetic data so protocols can
be exercised end-to-end without real sensor input.

| Profile | Behavior |
|---|---|
| `healthy_baseline` | HRV ~50ms steady, sleep ~7.5h, daily Z2 walk, balanced macros. |
| `recovery_deficit` | HRV trending down, sleep <6h, no workouts. Stress signal. |
| `overtraining` | HRV drops 20% from baseline, RHR up 5bpm, multiple high-intensity workouts. |
| `nutrition_gap` | All metrics normal, nutrition inputs return `null` (sim Cronometer-not-syncing). |
| `partial_signal` | Random subset of metrics return `null` to test graceful-degradation paths. |

## Fixture override

For exact-input testing, point `config.fixture` at a YAML file:

```yaml
# fixtures/morning-readiness-edge.yaml
hrv:
  latest:        { value: 32, unit: ms, timestamp: "2026-05-03T07:30:00-04:00" }
  mean(14d):     { value: 48, unit: ms, timestamp: "2026-05-03T07:30:00-04:00" }
sleep_total:
  latest:        { value: 5.4, unit: hours, timestamp: "2026-05-03T07:30:00-04:00" }
```

The mock returns these values verbatim. Useful for golden-file tests of
protocol output.

## Aggregations / windows

All canonical aggregations supported; the mock interprets the request
and synthesizes a value consistent with the profile.

## Authentication

None.

## Caveats

- **Synthetic, not real.** Output should never be persisted as if it
  were a real reading. The runtime SHOULD warn loudly when the active
  default adapter is `mock`.
- **Profile semantics may evolve.** Profile values are tuned for
  protocol-evaluation realism, not biological accuracy. PRs that argue
  a profile is producing implausible data are welcome.

## Reference implementation

Built into runtime libraries. Not a separate package — every runtime
that implements the adapter contract is expected to ship a mock.
