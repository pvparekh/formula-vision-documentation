# ADR 004: Production browser delivery contract

**Status:** Accepted

## Context

Formula Vision v2 canonical Parquet is optimized for reproducibility, validation and analysis, not direct browser consumption. Milestone 3 benchmarked real Bahrain 2024 race-window data across compact row JSON, column JSON, MessagePack and a custom fixed-width binary format using 15, 30, 60, 120 and 300 second position windows.

The corrected race-window benchmark contained 432,180 position rows, 417,180 car-telemetry rows and 30,506 timing samples.

At a 300-second position window:

- column JSON: 19 files, ~1.87 MiB gzip total, ~102 KiB first chunk, ~105 KiB p95 chunk;
- column MessagePack: 19 files, ~1.84 MiB gzip total, ~98 KiB first chunk, ~103 KiB p95 chunk;
- FVP1 binary: materially larger on the wire (~2.53 MiB gzip total);
- Chrome decoded the representative 300-second JSON and MessagePack chunks in roughly the same ~2.3 ms on the reference machine.

MessagePack therefore saved only tens of kilobytes across an entire race while requiring an additional runtime codec and making artifacts less inspectable.

## Decision

Use the following public web-delivery contract:

1. **Positions**: 300-second column-oriented JSON chunks.
   - integer race-relative millisecond timestamps;
   - compact driver indices;
   - integer X/Y coordinates;
   - one 20-driver snapshot at chunk start;
   - one 20-driver lookahead snapshot after chunk end;
   - each chunk is independently seekable and interpolation-complete.

2. **Timing**: one compact whole-race JSON product containing timestamp, driver, position, gap-to-leader and interval-ahead.

3. **Driver telemetry**: one whole-race column-oriented JSON product per driver, fetched lazily when a product surface needs it.

4. **Laps/results/events/weather/status**: small whole-race JSON products.

5. **Canonical Parquet**: internal analytical/rebuild layer only; never the browser protocol.

6. **Browser timeline**: race-relative milliseconds are authoritative. Legacy frame indices are exposed only as virtual 12.5 Hz UI coordinates for backward-compatible controls.

## Why JSON wins

- negligible transfer penalty versus MessagePack at this scale;
- native browser parsing;
- no codec dependency in the production replay path;
- directly inspectable in DevTools and object storage;
- simpler debugging, versioning and incident response;
- gzip/Brotli-friendly repetitive column structure;
- benchmarked decode cost is comfortably inside the latency budget.

## Consequences

Positive:

- only 19 position objects for a Bahrain-length race at 300-second windows;
- cold seek requires one small position object;
- timing and event queries become local binary-search operations after one small fetch;
- selected-driver telemetry is a single small lazy fetch;
- browser artifact format is independent from canonical analytical storage;
- public artifacts remain easy to inspect and validate.

Tradeoffs:

- JSON uses more uncompressed bytes than MessagePack;
- explicit compact field names and schema documentation are required;
- the browser replay runtime must materialize per-driver series from column payloads;
- future materially larger datasets may justify revisiting the encoding decision.

## Revisit criteria

Rebenchmark if one of these becomes true:

- typical position chunks exceed the latency/memory budget;
- CDN/object-request economics materially change;
- substantially richer per-sample fields are added to the hot position path;
- browser decode becomes a measurable frame-budget problem;
- mobile profiling materially disagrees with desktop Chromium results.

## Final driver-telemetry browser evidence

The representative full-race driver artifact measured approximately:

- column JSON: 132.62 KiB gzip, 1.7 ms decode, 24.0 ms cold fetch/decode;
- column MessagePack: 129.69 KiB gzip, 2.1 ms decode, 27.8 ms cold fetch/decode.

The ~2.2% compressed-size saving did not justify the extra runtime codec or lower inspectability. Column JSON is therefore accepted for driver telemetry as well as positions.
