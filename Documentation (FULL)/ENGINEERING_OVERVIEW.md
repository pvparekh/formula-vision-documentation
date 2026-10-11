# Formula Vision Engineering Overview

## Purpose

This document explains Formula Vision as one engineering system rather than as a list of features.

Formula Vision is a historical Formula 1 race replay and analysis product built from real timing, telemetry, race-control, weather, lap, tyre, pit, and classification data. The product began as a senior-year full-stack project and was later rebuilt around production-minded data engineering, reproducibility, delivery efficiency, correctness, and maintainability.

The current system combines:

- a typed canonical data platform;
- timing-authoritative race intelligence;
- browser-specific delivery artifacts;
- immutable object-storage publication;
- a catalog-driven historical archive;
- a client-owned replay engine;
- MAP and CHASE spatial visualization;
- operational automation, validation, quarantine, and rollback.

The most important engineering characteristic is the separation between **race truth**, **delivery optimization**, and **presentation**.

## Product at a glance

The browser product supports:

- multi-season race discovery through a promoted catalog;
- real circuit previews and race briefings;
- full-race play, pause, seek, and variable-speed playback;
- timing tower, gaps, intervals, laps, sectors, pits, tyres, status, weather, and results;
- race-control and semantic event context;
- selected-driver speed, RPM, gear, throttle, brake, and DRS telemetry;
- top-down MAP visualization and driver follow;
- perspective CHASE visualization with a separate ideal-guide layer and contextual turn information.

New historical races can be published through the pipeline without a frontend code change.

## Current architecture

```text
SOURCE / BUILD PLANE

FastF1 and upstream timing sources
              |
              v
residential acquisition boundary
              |
              v
canonical extraction
typed Parquet + manifest + lineage
              |
              v
canonical validation
              |
              v
race intelligence derivation
              |
              v
browser artifact build
              |
              v
browser validation
              |
              +---------- invalid ----------> quarantine report
              |
              v
immutable release planning
              |
              v
Cloudflare R2
content-addressed race releases
              |
              v
immutable race catalog
              |
              v
catalog/latest.json
compare-and-swap promotion


DELIVERY / CLIENT PLANE

Vercel frontend
      |
      v
catalog/latest.json
      |
      v
immutable catalog
      |
      v
immutable race manifest
      |
      + positions
      + timing
      + laps
      + intelligence
      + telemetry
      + events/results
      |
      v
client-owned replay clock
      |
      + MAP
      + CHASE
      + timing / telemetry / results UI
```

Historical replay does not need an always-on replay API.

## Why the original architecture changed

The original college project proved the core product idea: sparse source telemetry could be transformed into a smooth interactive race replay.

Its architecture was appropriate for a proof of concept:

```text
FastF1
  |
  v
Python preprocessing
  |
  v
large replay JSON
  |
  v
FastAPI + WebSocket
  |
  v
React replay
```

The browser interpolated between source samples using `requestAnimationFrame`, which remains one of the strongest ideas in the project.

The rebuild addressed different questions:

- Can historical races be replayed without keeping application compute alive?
- Can the UI be rebuilt without rebuilding race truth?
- Can race order come from authoritative timing instead of GPS-derived progress?
- Can bad sessions be isolated before users see them?
- Can hundreds of historical races be added through data operations instead of React edits?
- Can releases be reproduced, hashed, verified, rolled back, and incrementally extended?

The answer was to treat Formula Vision as a data product.

## Canonical data engineering

The canonical layer is a typed, normalized, versioned analytical snapshot stored in Parquet.

It preserves:

- session identity and source lineage;
- driver identity and team metadata;
- official results and classification;
- laps, sectors, tyre, stint, and pit information;
- timing-stream position, gap, and interval;
- track and session status;
- race-control messages;
- weather samples;
- X/Y/Z position telemetry;
- speed, RPM, gear, throttle, brake, and DRS telemetry;
- optional circuit turn metadata.

Canonical data is deliberately not shaped around one frontend component.

That design allows browser serialization, caching, visual design, and new analytical products to change without changing the durable race contract.

### Data authority

Formula Vision assigns authority by question.

| Product question | Authority |
| --- | --- |
| Race position | timing product |
| Gap to leader | timing product |
| Interval ahead | timing product |
| Official classification | results |
| Lap and sector facts | laps / race intelligence |
| Drawn car position | position telemetry |
| Speed / RPM / gear / controls | car telemetry |
| Turn numbers / markers | circuit metadata when available |

This boundary corrected a major v1 limitation: spatial progress is useful for drawing cars, but it is not a reliable replacement for official timing.

## Validation and failure behavior

The pipeline distinguishes warnings from hard validation failures.

Examples of validation include:

- identity consistency across datasets;
- monotonic high-frequency time series;
- timestamp normalization;
- telemetry sanity ranges;
- artifact hashes and byte sizes;
- browser manifest consistency;
- publication graph verification.

Invalid upstream values are normalized rather than exposed as plausible product values. For example, known unavailable/sentinel telemetry can become null with explicit provenance instead of appearing as a real measurement.

A race that cannot satisfy the contract is quarantined.

The archive can continue publishing unrelated valid races when explicitly configured to continue on quarantine.

This is a key design choice: **a smaller truthful archive is preferable to a larger archive containing plausible-looking incorrect races**.

## Race intelligence

Race intelligence is derived once from canonical products rather than repeatedly reconstructed in the browser.

The derived layer supports facts such as:

- current timing order;
- gap and interval context;
- position change;
- last and best lap;
- sector performance;
- fastest-lap progression;
- stint and tyre context;
- pit windows;
- classification state;
- race-control and status context.

The browser combines those products with the replay clock to present a coherent snapshot at any timestamp.

## Browser delivery contract

Canonical Parquet is optimized for durability and analysis, not direct browser playback.

The browser contract therefore separates access patterns:

- multi-driver position chunks for immediate spatial replay;
- compact timing state;
- laps and race intelligence;
- small status/control/weather products;
- official results;
- selected-driver telemetry loaded lazily;
- track and circuit metadata.

The contract was selected through delivery benchmarking rather than by choosing a serialization format first.

This separation creates two useful freedoms:

1. the canonical analytical schema can remain stable while browser encoding evolves;
2. the browser can fetch only the data needed for the current interaction.

## Client replay engine

The browser owns replay time.

Race timestamps are authoritative. Frame/chunk indices are storage locators.

The canonical replay hook:

- resolves the immutable race manifest;
- loads position chunks near the active timestamp;
- prefetches nearby chunks;
- keeps a bounded position cache;
- prepares timing and lap products;
- interpolates driver motion between source samples;
- uses `requestAnimationFrame` for display-rate updates.

The position cache is intentionally bounded rather than proportional to race length.

Detailed driver telemetry is fetched separately when a driver is selected.

This removes the old requirement for a Python process to pace historical frames over a WebSocket.

## MAP and CHASE spatial systems

### MAP

MAP is the stable whole-field renderer.

It provides:

- top-down circuit context;
- all-driver spatial replay;
- driver selection;
- driver-centered follow;
- bounded manual pan/zoom/reset;
- zoom-aware turn annotations;
- annotation collision avoidance around live driver markers.

### CHASE

CHASE is an additive perspective view.

The implementation separates three spatial concepts:

1. authoritative circuit progress;
2. filtered real vehicle lateral position;
3. a presentation-only ideal guide.

The car is not driven by the visible guide.

The road model uses circuit progress, tangent, lateral normal, curvature, and optional quality-gated elevation. The selected driver's source position is projected onto the circuit with continuity constraints so nearby or crossing sections do not cause arbitrary branch jumps.

The ideal guide is derived from circuit curvature and rendered ahead of the car as a restrained reference. It can improve visual reading without becoming timing authority or vehicle authority.

CHASE also includes:

- perspective-correct road width;
- a subtle rubbered groove;
- bounded speed/acceleration camera response;
- wheelbase-based chassis orientation;
- contextual upcoming-corner callouts placed in screen-space safe zones;
- fallback to MAP if spatial state cannot be resolved safely.

The spatial system is documented in [SPATIAL_CAMERA_ARCHITECTURE.md](SPATIAL_CAMERA_ARCHITECTURE.md).

## Circuit intelligence

Circuit information is shared between MAP and CHASE.

Canonical extraction attempts FastF1 circuit metadata when available, including turn number/letter, marker position, angle, distance, and rotation.

The frontend resolves that metadata into presentation annotations.

If a published legacy race does not contain circuit metadata, the frontend can infer approximate turn locations from circuit curvature. That fallback preserves usability while keeping official metadata authoritative when present.

Common names such as Eau Rouge, Raidillon, 130R, or Ascari are a curated display layer. Formula Vision does not invent unsupported corner names.

## Immutable publication

Race publication is content-addressed and immutable.

At a high level:

```text
validate
  -> plan
  -> read current pointer state
  -> publish or reuse immutable race objects
  -> read back and verify objects
  -> publish or reuse immutable catalog
  -> compare-and-swap catalog/latest.json
  -> verify promoted graph
```

The mutable latest pointer is promoted last.

This prevents partially published races from becoming discoverable and prevents a stale publisher from silently overwriting a newer catalog state.

Rollback promotes a previously verified catalog rather than deleting or rewriting race releases.

## Historical backfill and source acquisition

The backfill system discovers completed races, compares them with the promoted catalog, acquires missing source data when permitted, validates the race, builds browser products, publishes valid releases, and records quarantine reports.

Normal updates are missing-only.

A deliberate refresh mode can rebuild already-published sessions when a new canonical product, such as circuit metadata, requires source reacquisition and a new immutable release.

Full-season FastF1 acquisition is intentionally separated from generic cloud runners. Testing showed that upstream timing services can reject datacenter traffic and FastF1 can enforce a request-rate guard.

The production workflow therefore supports a labeled self-hosted Windows runner on a residential network.

## Delivery and cost model

The frontend is deployed independently from race data.

The current production topology is:

```text
Vercel
  |
  v
React/Vite application
  |
  v
public Cloudflare R2 race library
```

Immutable race releases and catalogs can receive long-lived cache headers. The small mutable catalog pointer is fetched without immutable caching.

The publication tooling includes a conservative 9 GB storage guardrail and can inspect actual stored bytes before mutation.

The architecture has near-zero idle compute for historical replay because the application is static-first and race data is object storage.

## Testing and operational controls

The repository includes automated coverage for:

- canonical normalization and validation;
- browser artifact generation;
- delivery benchmarking;
- race intelligence;
- hostile-race edge cases;
- publication semantics;
- R2 storage behavior;
- public delivery/CORS/cache behavior;
- historical backfill;
- repository text/style rules;
- frontend production builds;
- PowerShell operator script parsing.

Publication adds another level of defense through object read-back verification and catalog-graph verification.

## Scale and evidence

The project records concrete reference profiles instead of relying only on architecture claims.

Examples:

- Bahrain 2024 canonical snapshot: about **1.45 million rows**;
- reference snapshot: **49 manifest-tracked canonical artifacts**;
- reference canonical size: about **21.89 MiB**;
- high-frequency position and car telemetry account for roughly **98.5%** of canonical bytes;
- promoted historical catalog includes **170+ races** as of October 2026;
- archive publication preserves per-race quarantine rather than weakening validation to increase coverage.

These measurements influenced delivery benchmarking, lazy telemetry, chunking, storage budgeting, and archive operations.

## Engineering scope

Formula Vision demonstrates work across several boundaries that are often separated in production teams.

### Data engineering

- source normalization and typed analytical contracts;
- Parquet data modeling;
- data quality and lineage;
- idempotent historical backfill;
- quarantine and failure isolation;
- derived analytical products.

### Distributed delivery and operations

- content-addressed immutable publication;
- object-store verification;
- compare-and-swap pointer promotion;
- cache/CORS contracts;
- bounded storage budgets;
- self-hosted source acquisition;
- rollback and operational runbooks.

### Frontend and systems engineering

- bounded client caching;
- lazy data access;
- display-rate interpolation;
- hot-path updates outside React state;
- spatial projection and camera systems;
- responsive analytical UI;
- explicit loading, degraded, and failure states.

### Product engineering

- preserving data authority while improving presentation;
- designing an archive that scales without hard-coded races;
- separating exploratory visual systems from trusted fallbacks;
- treating motion and visual hierarchy as product behavior rather than decoration.

The project is strongest when evaluated as the integration of these concerns, not as any one isolated UI feature.

## Repository structure

```text
frontend/
    browser product, replay hooks, MAP/CHASE, telemetry and race UI

pipeline/v2/
    canonical extraction, validation, race intelligence,
    browser artifacts, publication, R2, backfill

docs/v2/
    architecture, engineering explanation, operations, ADRs,
    product design, data policy, milestone history

scripts/
    operator-facing PowerShell workflows for publication,
    historical backfill, delivery setup, and deployment

.github/workflows/
    CI plus self-hosted race-library publication orchestration

backend/
F1RaceViewer/
    preserved v1 implementation and project history
```

## Documentation model

The documentation follows reader intent:

- **README:** fast project orientation and setup;
- **Engineering Overview / Architecture:** current system explanation;
- **Runbooks:** operational procedures;
- **Reference docs:** data, delivery, benchmarks, policies;
- **ADRs:** significant architecture decisions and consequences;
- **Milestones:** frozen history of how the system evolved.

This keeps the root README scannable while preserving enough depth for engineering review and interviews.

## Current state and future work

The v2 frontend and race library are deployed, and the v1 production state is preserved as a rollback branch.

Near-term work remains intentionally product-focused:

- continue tuning CHASE camera smoothness;
- republish historical releases with official circuit metadata where useful;
- investigate quarantined races;
- continue completed-race archive expansion;
- finish cross-browser/mobile performance and accessibility QA.

Potential future product work includes battle comparison, strategy visualization, shareable race moments, and a separate live-mode architecture.

The historical replay architecture does not require those features to remain operational.
