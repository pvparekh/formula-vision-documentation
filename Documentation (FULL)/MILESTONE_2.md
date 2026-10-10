# Formula Vision v2: Milestone 2: Canonical Data Foundation

## Objective

Generate one clean, full-fidelity Bahrain 2024 Race dataset directly from FastF1 using a pinned source stack, normalize it into versioned canonical tables, validate it and produce a machine-readable build manifest.

Milestone 2 must not depend on the legacy precomputed/truncated replay JSON.

## Why this precedes more UI work

The current UI still contains POC approximations, including deriving a “positions behind leader” chart from position minus one, while the v1 replay position calculation sorted cars by GPS-derived cumulative distance.

FastF1 3.8.3 exposes the data needed for a stronger model: lap/sector timing, tyre/stint/pit data, car telemetry, position telemetry, weather, track/session status, race-control messages, official results, and timing stream samples with position/gap/interval.

The pipeline will isolate FastF1 behind an adapter so upstream changes do not become frontend rewrites.

## Layering

FastF1/timing sources -> source adapter -> canonical normalized tables -> validation -> web artifact builders -> R2/CDN/browser.

## Canonical identity

All records use stable session identity (`season`, `round`, `session_type`) plus driver number/abbreviation where applicable and session-time seconds for replay alignment.

## Validation classes

### Hard failures
- no session/results;
- non-monotonic per-driver time series;
- duplicate required keys;
- impossible types/ranges;
- missing required canonical datasets;
- artifact hash mismatch.

### Warnings
- one driver missing an optional channel;
- sparse weather;
- incomplete race-control data;
- unavailable timing gap stream;
- known upstream gaps that degrade a feature without corrupting the race.

Warnings stay in the build manifest and can later drive frontend feature availability.

## Definition of done

A fresh Bahrain 2024 build must answer from canonical data-not frontend heuristics:
- who every driver is;
- official grid/result/status;
- every driver's lap/sector times;
- tyres/stints/pit timing;
- timestamped position/gap/interval where available;
- track/session flags;
- race-control events;
- session weather;
- X/Y position samples;
- speed/RPM/gear/throttle/brake/DRS samples.

The next milestone benchmarks browser delivery formats from this canonical snapshot.

## Source acquisition topology

Hosted CI is deliberately not responsible for downloading full F1 timing data.
During the first Bahrain integration run, hosted GitHub infrastructure could
load basic results metadata but the timing/telemetry/weather/race-control
channels were unavailable.

Milestone 2 therefore separates:

1. **acquisition**: run from a network/runtime with legitimate source access or
   from a populated FastF1 cache;
2. **canonical transform + validation**: deterministic code that can be tested
   in hosted CI without network dependence.

The extractor fails explicitly when required full-fidelity source channels are
missing and publishes no canonical snapshot.

For the Bahrain reference build, run from the repository root:

```bash
python -m pip install -r pipeline/v2/requirements.txt
python pipeline/v2/extract_canonical.py --year 2024 --round 1 --session R
```

Successful output is written to
`artifacts/canonical/v1/2024/1/R/` and remains gitignored/internal.
