# Adapter: manual-csv

Reads from flat CSV files. Useful for metrics the user logs by hand —
body mass, training volume, qualitative notes — and for protocols that
want to incorporate sources without a dedicated API integration.

## Configuration

```yaml
adapters:
  manual-csv:
    config:
      files:
        body_mass: ~/Documents/health/body-mass.csv
        carbohydrates: ~/Documents/health/macros.csv
      timezone: America/New_York
```

| Field | Required | Description |
|---|---|---|
| `config.files` | ✓ | Map of canonical metric name → CSV file path. |
| `config.timezone` |   | Timezone for parsing date-only timestamps. Default: system tz. |

## CSV format

Each file MUST have a header row. The adapter looks for these columns
(case-insensitive):

| Column | Required | Notes |
|---|---|---|
| `date` or `timestamp` | ✓ | ISO 8601 date or datetime. Date-only assumes local-tz midnight. |
| `value` | ✓ | The reading. Must be numeric for scalar/aggregate metrics. |
| `unit` |   | If present, adapter converts to canonical. Otherwise assumes canonical. |
| `note` |   | Free-form annotation, surfaced in `raw_metadata`. |

For event-kind metrics (e.g., `workout`), the file has additional
required columns matching the event's fields:

```csv
timestamp,type,duration,hr_avg,hr_max,active_energy_burned
2026-05-03T20:29:33-04:00,Indoor Walk,3032,138.88,151,360
```

## Examples

### body_mass

```csv
date,value,unit,note
2026-04-28,286.6,lb,morning, post-bathroom
2026-05-01,285.0,lb,
2026-05-03,283.4,lb,
```

### carbohydrates

```csv
date,value,unit
2026-05-01,210,g
2026-05-02,180,g
2026-05-03,28,g
```

### workout

```csv
timestamp,type,duration,hr_avg,hr_max,active_energy_burned,intensity
2026-04-27T00:27:35Z,Functional Strength Training,5307,149.16,181,902.88,7.20
2026-04-27T21:43:36Z,Indoor Walk,6039,137.14,177,577.59,4.20
```

## Aggregations

The adapter loads the file, filters by window, then computes the
requested aggregation in-process. All canonical aggregations are
supported.

## Windows

Same as the canonical spec. Date-only timestamps are interpreted as
midnight in `config.timezone`.

## Authentication

None. File-based.

## Caveats

- **No live updates.** Adapter reads the file each request. For
  performance, the runtime may cache; the adapter declares no TTL.
- **Format drift.** If a header row changes, the adapter raises rather
  than silently misreading. CI tools (`tools/validate-sources`) can
  pre-check files at config-load time.
- **Unit specification matters.** Body mass logged in pounds without a
  `unit` column will be returned as kilograms (canonical) but with the
  *numeric value of pounds* — i.e., wrong. Always include the `unit`
  column unless values are already in canonical units.
- **No event-stream pivoting.** For event metrics, each row is one event;
  the adapter does not synthesize HR series from per-row aggregates.

## Reference implementation

Python: TBD. The format is simple enough that any spreadsheet, text
editor, or `python -c "import csv; ..."` script can author files
correctly.
