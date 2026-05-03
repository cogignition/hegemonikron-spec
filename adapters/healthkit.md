# Adapter: healthkit

Reads from Apple HealthKit on iOS and macOS. Native, zero-network, no
auth tokens. The default for any setup that runs on Apple hardware with
an Apple Watch.

## Configuration

```yaml
default_adapter: healthkit
```

That's all that's required. HealthKit authorization happens at
first-use via the OS prompt.

Optional fields:

```yaml
adapters:
  healthkit:
    metrics: [hrv, sleep_total, sleep_deep, sleep_rem, resting_heart_rate, body_mass, workout]
    config:
      hrv_kind: SDNN              # or RMSSD; default SDNN (Apple's default)
      sleep_includes_naps: false  # if true, includes daytime sleep in totals
```

## Canonical → native mapping

| Canonical metric | HealthKit identifier |
|---|---|
| `hrv` | `HKQuantityTypeIdentifier.heartRateVariabilitySDNN` |
| `resting_heart_rate` | `HKQuantityTypeIdentifier.restingHeartRate` |
| `heart_rate` | `HKQuantityTypeIdentifier.heartRate` |
| `sleep_total` | `HKCategoryTypeIdentifier.sleepAnalysis` (asleep stages) |
| `sleep_deep` | `HKCategoryTypeIdentifier.sleepAnalysis` value `asleepDeep` |
| `sleep_rem` | `HKCategoryTypeIdentifier.sleepAnalysis` value `asleepREM` |
| `sleep_core` | `HKCategoryTypeIdentifier.sleepAnalysis` value `asleepCore` |
| `sleep_awake` | `HKCategoryTypeIdentifier.sleepAnalysis` value `awake` |
| `sleep_in_bed` | `HKCategoryTypeIdentifier.sleepAnalysis` value `inBed` |
| `body_mass` | `HKQuantityTypeIdentifier.bodyMass` |
| `body_fat_percentage` | `HKQuantityTypeIdentifier.bodyFatPercentage` |
| `body_mass_index` | `HKQuantityTypeIdentifier.bodyMassIndex` |
| `step_count` | `HKQuantityTypeIdentifier.stepCount` |
| `active_energy` | `HKQuantityTypeIdentifier.activeEnergyBurned` |
| `basal_energy` | `HKQuantityTypeIdentifier.basalEnergyBurned` |
| `dietary_energy` | `HKQuantityTypeIdentifier.dietaryEnergyConsumed` |
| `protein` | `HKQuantityTypeIdentifier.dietaryProtein` |
| `carbohydrates` | `HKQuantityTypeIdentifier.dietaryCarbohydrates` |
| `total_fat` | `HKQuantityTypeIdentifier.dietaryFatTotal` |
| `fiber` | `HKQuantityTypeIdentifier.dietaryFiber` |
| `dietary_water` | `HKQuantityTypeIdentifier.dietaryWater` |
| `workout` | `HKWorkout` (event with HR series via `HKWorkoutRoute` / `HKQuantitySample`) |

Units are converted to canonical: HealthKit's pounds → kg for `body_mass`,
seconds → hours for `sleep_*`, etc.

## Aggregations

HealthKit's `HKStatisticsQuery` supports `discreteAverage`, `min`, `max`,
`cumulativeSum`, and `mostRecent`. The adapter maps:

| Canonical aggregation | HKStatisticsOptions |
|---|---|
| `latest` | `.mostRecent` (or last sample, depending on metric kind) |
| `mean` | `.discreteAverage` |
| `median` | computed in adapter from sample list (HealthKit doesn't provide native) |
| `min` | `.discreteMin` |
| `max` | `.discreteMax` |
| `sum` | `.cumulativeSum` |
| `count` | computed from sample list |
| `list` | raw `HKSample` array, mapped to canonical units |

## Windows

| Canonical window | HealthKit predicate |
|---|---|
| `today` | midnight-to-now in user's calendar timezone |
| `last_night` | most recent sleep window (sleep_in_bed start → wake) |
| `last_workout` | predicate matching the most recent `HKWorkout` |
| `Nh` / `Nd` / `Nw` / `Nm` | `HKQuery.predicateForSamples(withStart:end:options:)` |

## Authorization

Adapter must request authorization for every type it reads. Common
practice is to declare all types up front in `Info.plist`'s
`NSHealthShareUsageDescription` and request authorization lazily on first
metric request. Denied authorization returns `{ confidence: "missing" }`
rather than raising — protocols should degrade gracefully.

## Caveats

- **HRV interpretation:** HealthKit returns SDNN by default; many
  athlete-facing tools report RMSSD. Set `config.hrv_kind` explicitly if
  the protocol's thresholds assume one or the other.
- **Sleep stage availability:** stage breakdowns (`asleepDeep`, `asleepREM`)
  are only available on Apple Watch with watchOS 9+. Older devices report
  `asleep` only; the adapter returns `{ confidence: "estimated" }` when
  filling stage values from a single-bucket source.
- **Nutrition:** populated only if Cronometer or another tracker writes to
  HealthKit. Empty for users who don't track. Protocols that depend on
  nutrition metrics must handle `null` gracefully.
- **Workout HR series:** retrieved via `HKQuantitySample` queries scoped
  to the workout's date range; can be slow for long workouts. Adapter
  caches the series per workout id.

## Reference implementation

Swift: [`cogignition/hegemonikron`](https://github.com/cogignition/hegemonikron)
(when available — primary consumer).

For non-Swift clients on Apple platforms, HealthKit is only accessible
via Apple's frameworks; cross-language clients must shell out to a Swift
helper or use a bridge.
