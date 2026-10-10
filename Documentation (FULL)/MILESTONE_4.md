# Formula Vision v2: Milestone 4: Correct Race Intelligence

**Status: accepted for Bahrain baseline**

## Objective

Replace inherited frame-buffer approximations with race facts grounded in the
canonical timing, lap, result, status, weather and telemetry products.

Milestone 4 is a correctness milestone first. Visual-system redesign remains
Milestone 6.

## Authority model

- timing stream owns race position, gap-to-leader and interval-ahead;
- canonical laps own lap/sector times, compounds, tyre life and stint identity;
- official results own grid/classification/status/points;
- track/session status and race-control streams own race-state events;
- car telemetry owns speed/RPM/gear/throttle/brake/DRS;
- position telemetry owns only spatial placement.

The browser may select the authoritative record valid at the current replay
timestamp. It must not infer race facts from GPS ordering or rolling replay
buffers.

## Derived race-intelligence product

The pipeline publishes a small `intelligence.json` product containing:

- official grid and final classification facts by driver;
- per-driver best lap and best sectors;
- overall fastest lap;
- spoiler-safe fastest-lap progression events;
- stint summaries;
- pit-entry/pit-exit windows;
- position-change baselines;
- final status/points/laps-completed metadata.

Derived values retain explicit semantics. Pit duration derived from
`PitInTime -> PitOutTime` is labelled pit-lane duration, not stationary
pit-stop time.

## Frontend snapshot

At the current race-relative timestamp, the frontend composes an authoritative
snapshot from:

- timing.json;
- laps.json;
- intelligence.json;
- track_status.json;
- weather.json.

The snapshot drives the timing tower and top race state. It exposes current:

- position;
- gap to leader;
- interval ahead;
- lap;
- compound;
- tyre life;
- stint;
- pit-lane state;
- last lap;
- best lap so far;
- position change from grid;
- fastest lap so far;
- track status;
- weather.

## Delivery order

1. Derived intelligence pipeline + validation.
2. Timestamped race-intelligence hook.
3. Replace TimingTower approximations.
4. Replace TopBar race-state approximations.
5. Replace DriverDetail rolling-buffer statistics.
6. Lazy real driver telemetry UI.
7. Race event timeline primitives.
8. Edge-case race validation.

## Acceptance gates

- fastest lap is sourced from canonical lap timing, never leader transition
  timing;
- gap/interval values match the canonical timing stream;
- grid delta uses official grid position;
- tyre/stint state comes from canonical laps;
- pit state uses canonical pit timestamps;
- track status comes from canonical status transitions;
- driver telemetry uses the lazy per-driver telemetry artifact;
- no UI labels a pit-lane traversal duration as stationary pit-stop time;
- seeking backward rewinds all intelligence state correctly;
- seeking forward cannot reveal future fastest-lap/best-lap information;
- legacy races continue using their compatibility UI until migrated.


## Real-race acceptance workflow

After the branch gate is green, rebuild the already-extracted canonical race:

```bash
python pipeline/v2/build_web_artifacts.py --year 2024 --round 1 --session R
```

No FastF1 re-extraction is required when the canonical snapshot is unchanged.

Then print deterministic acceptance targets:

```bash
python pipeline/v2/inspect_race_intelligence.py --year 2024 --round 1 --session R
```

The inspector reports:

- final canonical fastest lap and its derivation basis;
- fastest-lap progression and a before/after rewind test;
- first pit-state transition and seek targets;
- track-status transitions;
- race-control banner targets;
- weather coverage;
- representative lazy telemetry artifact sizes/coverage.

Use those timestamps to prove the UI follows the replay clock and does not leak future state.


## Semantic notes

- FastF1 `TyreLife` is preserved as tyre life, not assumed to mean laps
  elapsed since the current race stint began. A reused tyre set can include
  mileage from a prior session.
- FastF1 result `Time` is winner elapsed time for P1 and a gap semantics
  value for trailing classified drivers. The existing canonical-v1 field name
  `race_time_seconds` is retained for schema stability; web presentation
  explicitly labels the value as Time / Gap. A future canonical schema version
  should rename the field rather than mutating canonical-v1 in place.


## Bahrain acceptance record

Bahrain 2024 is the accepted normal-race reference implementation for
Milestone 4.

Production evidence:

- all 1,107 non-deleted timed laps remained eligible for record progression;
- zero official timing exclusions were required;
- FastF1-official fastest lap resolved to VER, lap 39, 1:32.608;
- fastest-lap history remained replay-time safe: LEC 1:34.090 remained visible
  before VER completed lap 39 and changed to VER only after 63:18.633;
- seeking/playback around the change visually showed the expected LEC -> VER
  progression rather than leaking the eventual race fastest lap backward;
- track-status and race-control timestamps reconciled closely;
- field-wide telemetry quality was explicit, with only SAR gear below complete
  coverage at 99.7459%;
- all 18 source warnings were traced to upstream telemetry sentinel
  normalization, not race-order/lap-timing correctness;
- warning codes and contexts now propagate through the web manifest and
  acceptance inspector;
- migrated replay state never borrows future timing/lap/telemetry/status data.

Bahrain is intentionally not treated as sufficient evidence for the full race
library. The next checkpoint is a hostile-race validation matrix covering
wet/dry transitions, Safety Car/VSC, red flags, DNFs/lapped cars, unusual pit
sequences and sprint-weekend event structure before object-storage backfill.
