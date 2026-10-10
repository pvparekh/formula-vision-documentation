# Formula Vision v2: Product + Engineering Roadmap

## North star

Formula Vision should become an instant, no-login Formula 1 replay and analysis product that combines a production-minded motorsport data platform, a smooth race-replay engine, broadcast-quality analytical UI, shareable race moments/comparisons, and near-zero idle infrastructure cost.

The goal is not to maximize the number of technologies used. The goal is to make the architecture defensible, measurable, reliable, inexpensive and easy to explain.

## Engineering principles

- Preserve known-good rollback states during major migrations and production cutovers.
- Treat historical race data as immutable analytical data, not a live server workload.
- Treat race timestamps as authoritative; frame indices are storage locators only.
- Separate the canonical data model from the browser delivery format.
- Derive race facts once in the pipeline; the frontend presents them.
- Validate before publish. Never partially publish a session.
- Publish versioned immutable artifacts and switch manifests/catalogs atomically.
- Pin upstream dependencies used for reproducible race builds.
- Preserve lineage: source version, pipeline version, schema version and artifact hashes.
- Measure before choosing serialization, partition size or compression.
- Prefer simple stateless infrastructure over always-on services.
- Add complexity only when a product requirement earns it.

## Milestone 1: Stateless historical replay

**Status: accepted**

Proved that Bahrain 2024 can load and replay from static versioned chunks without a stateful FastAPI frame stream.

Key outcomes:
- deterministic 30-second replay chunks;
- schema-v2 manifest;
- SHA-256 artifact validation;
- browser-owned playback clock;
- timestamp-authoritative playback and seeks;
- bounded client cache and prefetch;
- static race catalog/metadata;
- legacy WebSocket fallback for unmigrated races;
- the v1 production state was preserved while the migration was validated.

The migration also exposed and documented two legacy data-quality issues: stale frame-count metadata and mixed 80/160 ms sampling caused by the old memory-reduction truncation workflow.

## Milestone 2: Canonical v2 data foundation

**Status: accepted**

### Goal

Stop building new functionality from the legacy replay JSON. Build one clean, full-fidelity Bahrain 2024 dataset directly from FastF1 and define the canonical tables/contracts every future race will use.

### Source contract

Initial pinned target:
- FastF1 3.8.3;
- pandas 2.3.3 (FastF1 3.8.3 requires pandas >= 2.1.1 and < 3);
- PyArrow 25.0.1 for canonical Parquet artifacts.

### Canonical datasets

- `session`: event/session identity and timing boundaries;
- `drivers`: number, abbreviation, name, team, team colour and classification metadata;
- `results`: grid, finish, status, points and official result fields;
- `laps`: lap/sector timing, position, validity, tyre/stint and pit timing;
- `timing_stream`: timestamped position, gap-to-leader and interval-to-ahead;
- `track_status`: green/yellow/SC/VSC/red state changes;
- `session_status`: started/finished/aborted/etc.;
- `race_control`: race-control messages and flags;
- `weather`: session-time weather samples;
- `position_telemetry`: X/Y/Z/on-track samples by driver;
- `car_telemetry`: speed/RPM/gear/throttle/brake/DRS samples by driver.

### Important correctness decision

Race order, gaps and intervals must not be inferred by sorting GPS-derived cumulative distance like v1. Timing data is the timing authority; GPS/position telemetry is for visualization.

### Canonical storage

Canonical analytical data should be typed columnar Parquet, not the final browser payload. This preserves a stable data contract while allowing browser serialization to evolve independently.

### Release gates

- clean Bahrain 2024 extraction completes from FastF1;
- no v1 truncation is involved;
- canonical tables have explicit schemas;
- timestamps normalize to numeric session-time seconds;
- driver identifiers reconcile across datasets;
- lap counts reconcile with classification within documented tolerances;
- high-frequency series are monotonic per driver;
- telemetry values pass sanity checks;
- manifest includes hashes, row counts, source versions and warnings;
- no canonical artifact is published if a hard validation fails.

## Milestone 3: Browser delivery benchmark

**Status: accepted**

Build the web payload from the clean canonical dataset instead of extending the legacy frame JSON.

Separate access patterns:
1. lightweight multi-driver replay positions;
2. timing state (position/gap/interval/lap/tyre);
3. event timeline;
4. selected-driver telemetry loaded lazily;
5. static analytics such as laps/stints/results.

Benchmark 15/30/60-second partitions and compact JSON+Brotli/gzip, MessagePack, and binary/Arrow-style representations where justified. Record compressed bytes, decode time, first-frame latency, cold/cached seek latency, request count, browser memory and CPU.

## Milestone 4: Correct race intelligence

**Status: accepted for the Bahrain baseline; hostile-race validation follows before M5**

Move race facts out of approximate frontend logic and into tested pipeline products:
- actual fastest lap and holder;
- accurate last/best lap and sectors;
- true gap to leader and interval ahead;
- starting grid and position change;
- compounds, tyre age and stint;
- pit entry/exit;
- SC/VSC/yellow/red status;
- race-control messages;
- session weather;
- classification/DNF status.

## Milestone 4B: Hostile-race validation

**Status: accepted**

Before object-storage publication or whole-season backfill, validate the accepted
Bahrain contracts against a deliberately hostile matrix: wet/dry transitions,
Safety Car/VSC, red flags, DNFs/lapped cars, unusual pit behavior and sprint-
weekend metadata. Failed sessions are quarantined with machine-readable reasons
rather than silently promoted.

## Milestone 5: Object storage + full-library automation

**Status: accepted**

Turn race availability into a pipeline operation.

The full-fidelity replay target is every race session from 2018 onward for which the required source data is available and validation passes. Pre-2018 can be represented separately with lower-fidelity results/history data later; it is not part of the telemetry-replay promise.

FastF1 -> canonical build -> validation -> browser artifact build -> validation -> R2 staging prefix -> atomic manifest/catalog promotion.

Requirements include immutable versioned keys, idempotent rebuilds, manual backfills, incremental season discovery, failed-race quarantine, and generated availability catalogs.

## Milestone 6: Product shell + visual system

**Status: accepted**

Season/race explorer, broadcast-grade timing tower, track-first replay workspace,
canonical Driver Focus, semantic event timeline, responsive layouts, keyboard
accessibility, honest loading states and explicit degraded/error states are now
implemented and accepted against the six-race hostile fixture set.

## Milestone 7: Comparative analysis features

Driver Focus and the semantic event timeline foundations are already part of the product shell. Milestone 7 should build higher-order analysis on top of those trusted products.

### Battle Mode
Select two drivers and synchronize track focus, true time gap, speed, throttle, brake, gear, DRS, lap delta and tyre/stint context.

### Strategy View
Show compound, stint length, tyre age, pit stops, SC/VSC context, and position before/after stops for the field.

### Race Moments
Derive high-confidence overtakes, pit-cycle changes, fastest-lap changes, and race-control moments from canonical intelligence so the replay can navigate meaningful events without inventing race facts.

## Milestone 8: Shareability and launch mechanics

Make interesting application state addressable by URL: race/session, replay timestamp, selected driver, comparison driver, active view and highlighted event.

Add dynamic titles/descriptions, OpenGraph cards, share-this-moment controls and a polished screenshot/video workflow. The viral loop should be “look at this exact race moment/comparison,” not “please visit my portfolio.”

## Milestone 9: Live mode beta

Keep live separate from historical replay. FastF1 3.8.x uses the newer SignalR Core endpoint and supports F1TV authentication, so live mode needs token handling, reconnect/resume, degraded-data behavior, recording and race-weekend operational testing.

Preferred shape: live feed -> recorder -> live state -> UI, while the recording feeds the normal post-session canonical pipeline.

## Milestone 10: Production hardening

**Status: in progress**

The v2 frontend is deployed and the historical library is served independently from the application build. Remaining hardening includes cross-browser acceptance, mobile/tablet behavior, Lighthouse/performance review, accessibility, structured client diagnostics, cost monitoring, source/publication review, and repeated rollback/smoke-test exercises.

## What makes this senior-level

The strongest signal is not feature count. It is the sequence of decisions: preserve a working production version while migrating; remove stateful compute from an immutable workload; detect inherited data-quality debt; establish canonical contracts; separate truth from presentation and canonical data from delivery optimization; pin/record upstream versions; design idempotent backfills and atomic publication; validate before publish; benchmark access patterns; and define release/rollback gates.

## Product differentiation

The 2026 F1-data ecosystem already contains tools with telemetry comparison, tyre strategy, timing towers, replay and live modes. Formula Vision should differentiate on the combination of instant/no-login browser experience, full-race spatial replay, polished race storytelling, exceptional battle/strategy visualization, shareable exact moments, a large race library, transparent data quality, extremely low infrastructure cost, and the documented evolution from student POC to production-minded data platform.

The project should feel like a motorsport product that happens to demonstrate excellent engineering-not a resume project wearing an F1 skin.