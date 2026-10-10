# Formula Vision v2: Milestone 3: Browser Delivery Benchmark

**Status: accepted**\n\n## Objective\n\nChoose the browser-delivery contract from measured product access patterns, not library preference.\n\nCanonical Parquet remains the stable analytical source of truth. Milestone 3 generates disposable web candidates and compares them against the existing Bahrain v1 replay artifact.\n\n## Product access patterns\n\n### Position replay\n\nRequired immediately when a race opens. It contains only session time, driver identity, X and Y.\n\nCandidates:\n- compact row JSON;\n- column-oriented JSON;\n- MessagePack;\n- custom fixed-width binary (FVP1).\n\n### Timing state\n\nSmall timing-driven product containing session time, driver, position, gap to leader and interval ahead.\n\n### Driver telemetry\n\nLoaded lazily by driver instead of eagerly for the entire grid: speed, RPM, gear, throttle, brake and DRS.\n\nOpening a race must not download every driver's telemetry trace.\n\n## Benchmark matrix\n\nRun the same Bahrain canonical snapshot at 15, 30 and 60 second position/timing partitions. The matrix runner performs all three and writes one comparison report.\n\nPrimary command:\n\n    python pipeline/v2/run_delivery_matrix.py\n\nThe lower-level benchmark command remains available for individual experiments.\n\nEach run records raw bytes, deterministic gzip bytes, median encode/decode time per chunk, file count, row count and the old Bahrain replay JSON size when present.\n\nPython timing is only an offline screening metric. The strongest 2-3 candidates move to a Vite/browser harness for real fetch/decode/seek/memory measurements.\n\n## Initial performance budgets\n\n- historical race launch has no backend dependency;\n- no full-race download before first playback;\n- first playable data approximately <= 1 second on normal broadband;\n- cached seek should feel instant;\n- cold seek normally < 1 second;\n- replay memory remains bounded independently of race duration;\n- driver telemetry downloads only when a feature actually needs it.\n\n## Definition of done\n\n- Bahrain produces candidate artifacts reproducibly;\n- JSON, MessagePack and custom binary are compared;\n- 15/30/60-second partitions are compared;\n- browser harness measures strongest candidates;\n- one position-delivery contract is selected and documented;\n- one lazy driver-telemetry contract is selected and documented;\n- the winning artifacts drive full Bahrain replay without the legacy frame JSON.

## Final gate before freezing the delivery contract

The first 15/30/60-second matrix narrowed the position candidates to
column-oriented JSON and MessagePack. The custom binary candidate must be
decoded into real usable arrays before comparing decode time; header-only
validation is not a fair benchmark.

Because the compressed position chunks are already very small, the matrix now
also tests 120-second and 300-second windows. Larger windows may reduce request
count substantially while still keeping first-load and cold-seek payloads well
inside the latency budget.

After regenerating the matrix, export representative fixtures:

```bash
python pipeline/v2/export_browser_benchmark.py
```

Then run the frontend development server:

```bash
cd frontend
npm run dev
```

Open:

```text
http://localhost:5173/benchmark-m3.html
```

The Chromium harness measures first-chunk fetch+decompression, materialized
decode time, and representative cold seeks for column JSON, MessagePack and
FVP1. It uses the generated local fixtures only; benchmark data is gitignored.

Milestone 3 is not complete until the winning position format drives Bahrain
end-to-end without the legacy frame JSON.


## Delivery contract decision: frozen

Browser evidence now freezes the production delivery contract:

- positions: 300-second column-oriented JSON chunks;
- timing: one whole-race compact JSON product;
- driver telemetry: one whole-race column-oriented JSON product per driver, loaded lazily;
- laps/results/weather/status/race-control: whole-race JSON products;
- canonical Parquet remains internal analytical truth.

The representative driver-telemetry browser benchmark measured:

- JSON: ~132.62 KiB gzip, ~1.7 ms decode, ~24.0 ms cold fetch/decode;
- MessagePack: ~129.69 KiB gzip, ~2.1 ms decode, ~27.8 ms cold fetch/decode.

The small transfer saving did not justify an additional runtime codec. Encoding research is closed unless future scale triggers the ADR revisit criteria.

## Explicit Milestone 4 boundary

The existing DriverDetail and some timing-derived UI still contain legacy frame-buffer approximations. Milestone 3 does not redefine those product calculations. It publishes the canonical products required to replace them.

Milestone 4 owns:
- canonical fastest/personal-best lap presentation;
- true gap and interval presentation;
- sector states;
- stint/tyre/pit intelligence;
- real driver telemetry UI (speed/RPM/gear/throttle/brake/DRS);
- event-aware race intelligence.

This prevents UI modernization from obscuring the Milestone 3 data-plane acceptance gate.

## Final Milestone 3 acceptance gate

Milestone 3 closes only after the real Bahrain 2024 canonical snapshot is built through `build_web_artifacts.py` and the normal application proves:

- v3 manifest is selected before v2/WebSocket fallback;
- no request for the legacy `2024_1_frames.json`;
- no Bahrain replay WebSocket;
- metadata is served from the v3 manifest;
- results are served from the v3 static artifact;
- initial replay starts from v3 timing/laps/position artifacts;
- play/pause and 0.5x/1x/2x/4x work;
- ±10 second seek works;
- scrub seeks work across multiple 300-second chunk boundaries;
- cars remain continuous at chunk boundaries;
- driver follow remains functional;
- race-end state is reached cleanly;
- no migrated-artifact failure is silently masked by the backend.

After those checks, the foundational delivery architecture is frozen and development moves to Milestone 4.


## Acceptance record

Bahrain 2024 was accepted end-to-end on the normal Formula Vision application after the production v3 artifact build.

Observed acceptance evidence:

- canonical build -> v3 browser artifact build completed successfully;
- 19 production position chunks for the 5,598.252 second race window;
- 432,180 race-window position rows;
- 20 drivers;
- v3 manifest/timing/laps/position chunks loaded by the normal application;
- legacy `2024_1_frames.json` was absent from Network requests;
- no Bahrain replay WebSocket appeared while the race was playing;
- static v3 results rendered successfully;
- track/cars rendered correctly;
- play/pause, playback-speed controls, seeking/chunk-boundary behavior, driver-follow, race-end behavior and race switching were manually accepted.

The historical replay data plane is therefore frozen. New product correctness work belongs to Milestone 4.
