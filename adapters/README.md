# Adapters

An adapter translates requests in the canonical Hegemonikron vocabulary
into native queries against a backing store, and returns values in
canonical units. Adapters are how protocols stay portable across stacks.

## The contract

Every adapter, regardless of language or platform, MUST honor this
behavioral contract:

### 1. Accept a metric request

```
{
  metric:       "hrv",                   # canonical name (vocabulary.yaml)
  aggregation:  "mean",                  # one of vocabulary aggregations
  window:       "14d" | "today" | ...,   # symbolic or relative
  field:        "duration"               # only for event-kind metrics
}
```

### 2. Return a result in canonical units

```
{
  value:      <number | array | object>,   # depends on aggregation + kind
  unit:       "ms",                        # canonical unit from vocabulary
  timestamp:  <iso8601>,                   # when the value was observed
                                           # (for aggregations: end of window)
  source:     "influxdb",                  # adapter name
  confidence: "exact" | "estimated" | "missing",
  raw_metadata: { ... }                    # adapter-specific provenance
}
```

### 3. Convert units to canonical

If the native store uses different units (e.g., body mass in pounds when
canonical is kilograms), the adapter converts before returning. Apps
never see native units.

### 4. Handle missing data gracefully

Empty or absent data returns `{ value: null, confidence: "missing" }`,
not an error. Hard failures (network down, auth invalid) raise.

### 5. Declare supported metrics + aggregations

Each adapter publishes a capabilities manifest (see individual adapter
docs) so runtimes can route requests to adapters that actually support
the request, and surface clear errors when no adapter does.

## Configuration shape

User-side configuration validates against
[`schema/sources.schema.json`](../schema/sources.schema.json). Minimum:

```yaml
default_adapter: healthkit
```

Mixed setups (different adapters for different metrics):

```yaml
default_adapter: healthkit
adapters:
  influxdb:
    endpoint: ${HEGEMONIKRON_INFLUX_URL}
    metrics: [carbohydrates, protein, total_fat, dietary_energy]
    config:
      bucket: health
      org: kenning
      auth_env: INFLUXDB_TOKEN
```

Per-metric overrides for edge cases:

```yaml
metric_overrides:
  body_mass:
    adapter: manual-csv
    config:
      file: ~/Documents/weight-log.csv
```

## Authoring a new adapter

1. Drop a markdown file in this directory: `adapters/<name>.md`.
2. Document, at minimum:
   - Configuration parameters (with examples)
   - Default canonical-to-native mapping table
   - Supported metrics + aggregations
   - Auth model (env vars only — never literal tokens)
   - Caveats / known limitations
   - Reference implementation pointer (link to a working adapter)
3. Open a PR. Inclusion in this directory means "this adapter is part of
   the spec ecosystem"; it does not require a reference implementation
   in any specific language, but a link to one is strongly preferred.

## Currently documented adapters

- [`healthkit.md`](healthkit.md) — Apple Health on iOS / macOS
- [`influxdb.md`](influxdb.md) — InfluxDB 2.x time-series store
- [`manual-csv.md`](manual-csv.md) — flat CSV files (good for body mass, custom logs)
- [`mock.md`](mock.md) — test adapter; deterministic synthetic data

Adapters that are conceptually planned but not yet documented:
Garmin Connect, Whoop, Oura, Strava, Withings, Cronometer (direct API).
PRs welcome.
