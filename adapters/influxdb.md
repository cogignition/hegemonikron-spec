# Adapter: influxdb

Reads from an InfluxDB 2.x time-series database. The default backing
store for setups that ingest from Health Auto Export (HAE) or similar
into a self-hosted bucket.

## Configuration

```yaml
adapters:
  influxdb:
    endpoint: ${HEGEMONIKRON_INFLUX_URL}
    auth_env: INFLUXDB_TOKEN
    config:
      bucket: health
      org: kenning
      hrv_kind: SDNN              # adapter-level convention
      timezone: America/New_York  # for symbolic windows
```

| Field | Required | Description |
|---|---|---|
| `endpoint` | ✓ | Base URL (e.g., `http://192.168.97.2`, `https://influx.example.com`). |
| `auth_env` | ✓ | Env var name holding the token (never the literal token). |
| `config.bucket` | ✓ | Bucket name (e.g., `health`). |
| `config.org` | ✓ | Organization name (e.g., `kenning`). |
| `config.timezone` |   | IANA timezone for symbolic windows (default: system tz). |
| `config.hrv_kind` |   | `SDNN` (default) or `RMSSD`; affects label only — assumes ingest is consistent. |
| `config.measurement_overrides` |   | Override default measurement-name mapping (advanced). |

## Default measurement / field layout

The default mapping assumes ingest from Health Auto Export via a
HAE-bridge that writes:

- `health_metric` measurement with tags: `name`, `source`, `units`;
  field: `qty` (for scalar metrics) or `hr_avg` / `hr_max` / `hr_min`
  (for HR-derived rollups).
- `workout` measurement with tags: `id`, `name`, `source`; fields:
  `duration`, `active_energy_burned`, `hr_avg`, `hr_max`, `hr_min`,
  `intensity`.
- `workout_hr`, `workout_energy`, `workout_steps`, `workout_hr_recovery`
  for per-workout time-series detail.

If a deployment uses different schemas, override via
`config.measurement_overrides`:

```yaml
config:
  measurement_overrides:
    hrv:
      measurement: my_hrv_table
      tag_filters:
        device: garmin
      field: value
```

## Canonical → native mapping (default)

Each row below is the Flux template the adapter expands per request.

### Scalar metrics

| Canonical | Flux filter |
|---|---|
| `hrv` | `r._measurement == "health_metric" and r.name == "heart_rate_variability" and r._field == "qty"` |
| `resting_heart_rate` | `r._measurement == "health_metric" and r.name == "resting_heart_rate" and r._field == "qty"` |
| `heart_rate` | `r._measurement == "health_metric" and r.name == "heart_rate" and r._field == "hr_avg"` |
| `body_mass` | `r._measurement == "health_metric" and r.name == "weight_body_mass" and r._field == "qty"` (units: lb → kg) |
| `body_fat_percentage` | `r._measurement == "health_metric" and r.name == "body_fat_percentage" and r._field == "qty"` |
| `body_mass_index` | `r._measurement == "health_metric" and r.name == "body_mass_index" and r._field == "qty"` |

### Sleep stages

```flux
from(bucket: "health")
  |> range(start: ...)
  |> filter(fn: (r) => r._measurement == "health_metric" and r.name == "sleep_analysis")
  |> pivot(rowKey: ["_time"], columnKey: ["_field"], valueColumn: "_value")
  |> keep(columns: ["_time", "totalSleep", "core", "deep", "rem", "awake", "inBed"])
```

The adapter selects the appropriate column for the requested canonical
metric (`sleep_total` → `totalSleep`, `sleep_deep` → `deep`, etc.).

### Aggregate metrics (nutrition, activity)

| Canonical | Flux filter |
|---|---|
| `dietary_energy` | `r._measurement == "health_metric" and r.name == "dietary_energy" and r._field == "qty"` |
| `protein` | `r._measurement == "health_metric" and r.name == "protein" and r._field == "qty"` |
| `carbohydrates` | `r._measurement == "health_metric" and r.name == "carbohydrates" and r._field == "qty"` |
| `total_fat` | `r._measurement == "health_metric" and r.name == "total_fat" and r._field == "qty"` |
| `fiber` | `r._measurement == "health_metric" and r.name == "fiber" and r._field == "qty"` |
| `step_count` | `r._measurement == "health_metric" and r.name == "step_count" and r._field == "qty"` |
| `active_energy` | `r._measurement == "health_metric" and r.name == "active_energy" and r._field == "qty"` |

### Workouts (event metric)

```flux
from(bucket: "health")
  |> range(start: ...)
  |> filter(fn: (r) => r._measurement == "workout")
  |> pivot(rowKey: ["_time", "id", "name"], columnKey: ["_field"], valueColumn: "_value")
  |> keep(columns: ["_time", "name", "duration", "active_energy_burned",
                    "hr_avg", "hr_max", "hr_min", "intensity"])
  |> group()
  |> sort(columns: ["_time"], desc: true)
```

Returned shape:

```json
{
  "value": [
    { "time": "2026-05-03T00:29:33Z", "type": "Indoor Walk",
      "duration": 3032, "hr_avg": 138.88, "hr_max": 151,
      "active_energy_burned": 360, "intensity": 4.44 }
  ],
  "unit": "event",
  "confidence": "exact"
}
```

## Aggregations

| Canonical | Flux |
|---|---|
| `latest` | `\|> last()` (or `\|> sort(desc: true) \|> limit(n: 1)` for pivoted streams) |
| `mean` | `\|> aggregateWindow(every: 1d, fn: mean)` then `\|> mean()` over window |
| `median` | computed from `\|> sort()` + index, or via `\|> quantile(q: 0.5)` |
| `min` | `\|> min()` |
| `max` | `\|> max()` |
| `sum` | `\|> sum()` (cumulative) |
| `count` | `\|> count()` |
| `list` | no aggregation; rows returned as-is, mapped to canonical units |

## Windows

The adapter applies `import "timezone"; option location =
timezone.location(name: config.timezone)` so symbolic windows resolve in
the user's local time:

| Canonical | Flux |
|---|---|
| `today` | `range(start: today())` |
| `last_night` | `range(start: -36h)`, then filter to `name == "sleep_analysis"`, take most recent |
| `last_workout` | `range(start: -7d)`, take most recent `workout` event |
| `Nh` / `Nd` / `Nw` / `Nm` | `range(start: -<N><h\|d\|w\|m>)` |

## Authentication

Two paths, depending on deployment:

**Path A (kubectl exec).** Adapter runs `kubectl exec -i -n <ns> <pod>`,
pipes Flux via stdin. Auth comes from kubeconfig; no token required from
this adapter. Requires the runtime to have kubectl access; intended for
host-side automation.

**Path B (HTTP API).** Adapter sends Flux to `POST {endpoint}/api/v2/query`
with `Authorization: Token ${INFLUXDB_TOKEN}` and any required Cloudflare
Access service-token headers. Required for runtimes without cluster access
(iOS, web, frontier-model sessions in cloud sandboxes).

`config` may include a `path` field to select between the two:

```yaml
config:
  path: A   # "A" (kubectl exec) | "B" (HTTPS); default: B if endpoint scheme is https
```

## Caveats

- **Schema assumptions.** The default mapping assumes Health Auto Export
  ingest. Other ingesters need `measurement_overrides`.
- **Tag-key consistency.** Flux is strict about tag value casing. If
  ingest writes `Sleep Analysis` (capitalized) and queries assume
  `sleep_analysis`, results are silently empty. Verify with
  `schema.tagValues(tag: "name")` during onboarding.
- **HRV kind.** Apple Health and Garmin use SDNN; Whoop and many athletic
  tools use RMSSD. The adapter does not convert; it labels. Make sure
  ingest and protocol thresholds agree.
- **Path A on cloud routines.** kubectl from a frontier-model session in
  a cloud sandbox requires the host-shell escape (osascript on macOS).
  Path B is preferred for cloud-side use; see CLAUDE.md in
  `cogignition/hegemonikron-ops` for the bridge details on a maintainer-style
  setup.

## Reference implementation

Bash + kubectl reference: [`cogignition/hegemonikon-ops`](https://github.com/cogignition/hegemonikon-ops)
(`bin/snapshot-hegemonikon.sh` is a Path A consumer of this adapter
contract; not a complete adapter, but illustrates the query shape).

Swift HTTP adapter: [`cogignition/hegemonikron`](https://github.com/cogignition/hegemonikron)
(when available; will implement Path B for the iOS/macOS app).
