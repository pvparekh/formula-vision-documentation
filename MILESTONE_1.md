# Formula Vision v2: Milestone 1: Stateless Historical Replay

## Goal

Prove that a complete historical Formula 1 race can be replayed without a
stateful server streaming every frame.

The public v1 deployment remains unchanged on `main`. All milestone work lives
on `v2/milestone-1-static-replay` until it passes the release gates below.

## Why this is first

The v1 backend loads a full race JSON file into Python memory and paces
individual frames over one WebSocket connection. That was a reasonable
free-tier proof-of-concept architecture, but the race data is immutable and
already precomputed.

Milestone 1 preserves the current `Frame` contract:

```text
{ t, lap, drivers: { CODE: { x, y, pos } } }
```

This lets the existing SVG renderer, timing tower, camera follow, battle lines,
controls and driver-history UI survive the transport migration.

## New data path

```text
v1 precomputed race JSON
        ↓
source validation
        ↓
deterministic 30-second chunks
        ↓
manifest + SHA-256 checksums
        ↓
static files
        ↓
browser loader
        ↓
bounded cache + prefetch
        ↓
client-owned playback clock
        ↓
existing 60 FPS interpolation
```

## Legacy data-quality findings discovered during migration

The milestone immediately exposed two inherited v1 artifact inconsistencies.

### 1. Stale frame-count metadata

Bahrain's metadata claims 69,979 frames while the committed frame artifact
contains 48,985. The difference comes from the v1 memory-reduction workflow:
`truncate_race.py` rewrote the large frame file but did not regenerate its
metadata.

v2 does not silently trust either value. The manifest records both values,
preserves the discrepancy as a structured data-quality warning, and uses the
actual immutable frame artifact as authoritative for migration.

### 2. Variable sampling cadence

The extractor originally resampled at 25 Hz and `precompute.py` kept every
second frame, creating approximately 12.5 Hz / 80 ms spacing. The later v1
truncation kept the race start and finish at that cadence but sampled the middle
more aggressively.

For Bahrain v1, the migrated source contains:

- 27,990 intervals at approximately 80 ms;
- 20,994 intervals at approximately 160 ms.

The old WebSocket server also paced frames with a fixed `0.1 / speed` delay,
which was independent of the timestamps stored in the data.

v2 therefore treats **race timestamps as authoritative**. Playback position,
variable-speed timing, interpolation duration and time-based seek operations are
derived from actual frame timestamps instead of assuming a constant frame rate.
Frame indices remain useful storage/UI locators but no longer define race time.

## Browser runtime

For a migrated race, the frontend:

1. requests `/data/v2/{year}/{round}/manifest.json`;
2. loads only the chunk containing the required frame;
3. prefetches the next chunk;
4. retains a bounded five-chunk cache;
5. calculates replay position from `performance.now()`, source cadence and
   playback speed;
6. updates the unchanged `currentFrameRef` consumed by the rest of the UI.

A clean 404 means a race has not migrated yet, so the development branch falls
back to the original WebSocket transport.

## Release gates

Milestone 1 is complete only when Bahrain 2024 passes all of these:

- [x] pipeline unit tests pass
- [x] full source race passes validation with legacy quality debt explicitly recorded
- [x] chunks cover every source frame exactly once
- [x] chunk checksums validate
- [x] React/TypeScript production build passes
- [ ] frame 0 loads from static data
- [ ] play/pause works
- [ ] 0.5×, 1×, 2× and 4× work
- [ ] timeline seek works across chunk boundaries
- [ ] ±10-second seek works across chunk boundaries
- [ ] race reaches the final frame and ended state
- [ ] camera follow, timing tower and track rendering still work
- [ ] non-migrated races still fall back to v1 WebSocket delivery
- [ ] `main` and the public v1 deployment remain untouched

## Current generated reference artifact

The validated Bahrain 2024 reference build currently contains:

- schema version: 2
- source frames: 48,985
- race duration from timestamps: 5,598.24 seconds
- static partitions: 187
- partition window: 30 seconds
- compact JSON chunk bytes: approximately 62.56 MB
- bounded browser cache target: five chunks
- data-quality status: warning, with both inherited v1 issues documented

The current JSON size is intentionally **not** treated as the final storage
format. Compression/serialization benchmarking belongs to the next architecture
step after functional stateless replay is proven.

## Deliberately deferred

- richer speed/throttle/brake/gear/DRS telemetry
- canonical fastest-lap, lap, sector and gap calculations
- R2 publishing
- static session catalog/results
- all-race automation
- strategy and battle-comparison UI

Those features come after the delivery architecture is proven.
