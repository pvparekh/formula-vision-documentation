# Formula Vision v2: Initial Delivery Benchmark Findings

**Historical benchmark record.** The final browser contract is frozen in [ADR 004](adr/004-browser-delivery-contract.md). The remaining-gate language below is preserved as evidence of the decision process.

## Reference inputs

- Bahrain 2024 Race canonical snapshot
- 719,880 position rows
- 696,380 car-telemetry rows
- 30,536 timing samples
- canonical data-quality errors: 0
- all declared Bahrain feature domains available

## Position stream: first 15/30/60-second matrix

| Window | Encoding | Files | Full-race gzip MiB | Median Python decode/chunk ms |
|---:|---|---:|---:|---:|
| 15s | row JSON | 623 | 3.98 | 0.623 |
| 15s | column JSON | 623 | 2.39 | 0.491 |
| 15s | column MessagePack | 623 | 2.27 | 0.118 |
| 15s | FVP1 binary | 623 | 3.35 | invalid initial decode comparison |
| 30s | row JSON | 312 | 3.96 | 1.199 |
| 30s | column JSON | 312 | 2.33 | 0.951 |
| 30s | column MessagePack | 312 | 2.22 | 0.238 |
| 30s | FVP1 binary | 312 | 3.38 | invalid initial decode comparison |
| 60s | row JSON | 156 | 3.94 | 2.447 |
| 60s | column JSON | 156 | 2.28 | 1.825 |
| 60s | column MessagePack | 156 | 2.19 | 0.457 |
| 60s | FVP1 binary | 156 | 3.42 | invalid initial decode comparison |

### Immediate conclusions

1. Row-oriented JSON is eliminated for the high-frequency position product.
2. Column JSON and column MessagePack are the production finalists.
3. FVP1 does not currently justify itself on transfer size. Its original ~0.0015 ms decode result was invalid because the benchmark only validated header/length rather than materializing position arrays. The benchmark has been corrected.
4. Total compressed position size changes only modestly from 15s -> 60s while request count drops from 623 -> 156. This means request count and per-seek payload size are now more important than whole-race bytes.
5. The next matrix extends chunk windows to 120s and 300s and records first/P95/max compressed chunk size.

## Timing product

The 60-second experiment produced roughly 154 KiB compressed total across 97 files. Timing is therefore small enough that a whole-race timing/event product should be considered instead of hundreds of tiny requests. One parsed timing array also makes time-based binary search and arbitrary seeking straightforward.

## Driver telemetry

All 20 drivers together compress to roughly:

- column JSON: 3.52 MiB
- column MessagePack: 3.49 MiB

That is only about 179 KiB compressed per driver on average. The likely production access pattern is therefore:

`select driver -> fetch that driver's full-race telemetry once`

rather than chunking every driver's telemetry by replay time.

JSON and MessagePack have almost identical compressed size for this product, so the final choice should prioritize real browser decode cost, implementation complexity and dependency/runtime cost.

## Canonical source quality

The reference manifest contains zero hard validation errors. Its 18 warnings are source-sentinel normalization notices: 51,686 throttle samples and 53 gear samples were recognized as invalid upstream values and normalized to null before product delivery.

These warnings do not block replay, but future manifests should expose per-channel coverage percentages in addition to boolean feature availability.

## Production contract hypothesis before browser benchmark

- **positions:** chunked column JSON or MessagePack, likely larger than 60 seconds if browser measurements stay comfortably inside latency budgets;
- **timing/events:** whole-race compact product loaded once;
- **driver telemetry:** whole-race per-driver product loaded lazily;
- **laps/results/strategy/weather/status:** whole-race small analytical/event products;
- **canonical Parquet:** internal analytical/rebuild layer only, not browser protocol.

## Remaining Milestone 3 gates

1. Run 15/30/60/120/300-second corrected matrix.
2. Run Chromium benchmark on representative first/middle/last chunks.
3. Select final position encoding/window from real browser fetch+inflate+decode data.
4. Select driver-telemetry encoding using real browser decode data.
5. Generate production-style manifests/artifacts from canonical Bahrain.
6. Replay Bahrain end-to-end with no dependency on legacy `2024_1_frames.json`.
7. Validate play/pause/seek/speed/end-state and existing visualization behavior.

Only after those gates is the delivery contract frozen and Milestone 3 complete.