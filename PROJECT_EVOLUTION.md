# Formula Vision: Project Evolution

## Overview

Formula Vision has effectively been built twice.

The first version was a senior-year college project focused on proving that real Formula 1 telemetry could become a smooth, interactive race replay in a browser.

The second version began after revisiting the project with professional data-engineering and software-engineering experience. Instead of adding features to the original architecture, the project was rebuilt around data correctness, reproducibility, validation, static delivery, safe publication, archive scale, and a stronger product experience.

## 1. The original college project

The original goal was to take real historical Formula 1 telemetry, transform it into replayable race state, and make a complete race feel live in the browser.

v1 used FastF1, Python, pandas, NumPy, precomputed JSON, FastAPI, WebSockets, React, TypeScript, SVG, requestAnimationFrame interpolation, Framer Motion, and Recharts.

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
    React client
      |
      v
    smooth browser replay

The original project supported full-race playback, play/pause/seek, variable speed, telemetry-generated circuit shapes, full-field markers, top-down follow, timing, fastest-lap presentation, driver detail, results, and keyboard controls.

Its most important client technique was separating source sample cadence from display cadence. The browser interpolated between source samples every animation frame, producing smooth motion without changing race truth.

## 2. Why a rebuild was needed

The original system had normal first-project constraints:

- immutable historical replay was still stateful at runtime;
- the main race JSON was application-specific rather than a durable analytical model;
- ordering was inferred from spatial progress rather than official timing;
- the backend introduced memory, cold-start, and concurrency constraints;
- migration exposed inherited metadata and sampling inconsistencies.

Those limitations became the design input for v2.

## 3. Reframing the project

After gaining professional data-engineering experience, Formula Vision was approached as a data product.

The rebuild followed a few rules:

1. Race truth belongs in the pipeline.
2. Historical releases should be immutable.
3. Canonical analytical data and browser delivery data should be separate.
4. Published races should carry lineage and validation evidence.
5. Bad races should be isolated instead of silently weakened.
6. Publication should be atomic and reproducible.
7. Browser compute should be spent on interaction and presentation.
8. Infrastructure complexity should exist only when the product needs it.

## 4. Milestone 1: stateless historical replay

Branch: v2/milestone-1-static-replay

Bahrain 2024 was converted from the legacy replay format into deterministic static chunks with manifest identity, SHA-256 checks, timestamp authority, bounded caching, prefetch, and a client-owned replay clock.

This proved historical replay did not require a WebSocket server.

## 5. Milestone 2: canonical data foundation

Branch: v2/milestone-2-data-foundation

A typed canonical dataset was built directly from FastF1.

It preserves session, drivers, results, laps/sectors, timing, status/control, weather, X/Y/Z position telemetry, and detailed car telemetry.

Parquet became the analytical/rebuild layer.

The major correctness decision was to make official timing authoritative for position, gap to leader, and interval ahead.

## 6. Milestone 3: browser delivery benchmark

Branch: v2/milestone-3-delivery-benchmark

The project benchmarked what the browser should fetch rather than treating canonical storage as the delivery protocol.

The final contract separated high-frequency positions, timing, lazy selected-driver telemetry, and smaller race products.

## 7. Milestone 4: race intelligence

Branch: v2/milestone-4-race-intelligence

Race facts moved out of frontend approximations and into tested pipeline products: lap/sector state, fastest-lap progression, position change, stint/tyre context, pits, classification, weather, and race-control context.

## 8. Milestone 4B: hostile-race validation

Branch: v2/milestone-4b-edge-case-validation

The pipeline was tested against difficult races containing wet conditions, Safety Car/VSC, red flags, DNFs, lapped cars, sprint metadata, nonstarters, disqualification, and sparse telemetry.

This established that official/session drivers, spatially available drivers, and telemetry-available drivers are related but not assumed identical.

## 9. Milestone 5: immutable publication and archive scaling

Branches:

- v2/milestone-5-object-storage
- v2/milestone-5c-library-scaling

Race releases became content-addressed and immutable.

The ordered race library became an immutable catalog.

catalog/latest.json became the small mutable pointer and is promoted last with compare-and-swap semantics.

Published objects are read back and verified.

The same publication core was proven locally and against Cloudflare R2.

The historical backfill can discover races, acquire missing canonical data, validate, build browser products, publish valid races, quarantine failures, enforce storage budgets, and resume missing-only increments.

Cloud-hosted full-season source acquisition proved unreliable because upstream services can reject datacenter traffic, so the full-fidelity source boundary moved to a residential-network machine.

## 10. Archive expansion

The original promoted fixture contained the hostile/reference races used to validate the pipeline.

A 2018 backfill then exercised the library at larger scale while preserving quarantine rather than lowering validation standards.

Modern seasons were added through the same guarded path.

By the 2025 publication run, the promoted catalog reached 85 races. Twenty-two 2025 races were published while two remained quarantined for later investigation. The promoted graph was independently verified.

Completed 2026 races can be added incrementally through the same missing-only workflow.

## 11. Milestone 6: product shell

Branch: v2/milestone-6-product-shell

Once the data platform stabilized, work returned to product experience.

M6 introduced catalog-driven race discovery, race switching, timing authority, timing tower, track-first workspace, Driver Focus, semantic timeline, race control, results, explicit loading/error states, and responsive/keyboard behavior.

## 12. Visual-system rebuild

The UI was then redesigned around a motorsport-specific system called Telemetry Cinema:

- quiet chrome;
- expressive race content;
- condensed display typography;
- monospaced timing typography;
- fewer generic cards;
- real circuit geometry as the dominant visual motif;
- motion tied to race meaning.

The archive became season shelves. Race cards lazily resolve real circuit geometry and animate it in perspective on hover/focus.

Season rows gained contextual floating carousel controls.

The pre-race modal became a cinematic race briefing.

The viewer gained quieter navigation, an integrated transport dock, contextual race control, a refined timing rail, denser driver inspection, and a stronger track hierarchy.

## 13. Spatial CHASE camera

The stable top-down MAP renderer remains the default.

CHASE is a second additive camera built around the track as a coordinate system.

Each circuit becomes a closed spatial model containing progress, tangent, lateral normal, curvature, and optional elevation.

Driver position is projected onto that model with continuity constraints so close/crossing geometry does not cause arbitrary branch jumps.

The spatial system later separated authoritative progress, filtered source-derived lateral vehicle position, and a presentation-only ideal guide. This removed the conceptual coupling between the visible racing guide and the actual car path.

A shared circuit-intelligence layer now supplies zoom-aware MAP turn annotations and safe-side CHASE corner callouts. FastF1 circuit markers are used when published; legacy releases can fall back to geometry-derived turn inference.

CHASE uses perspective projection and a world-width track ribbon so nearby road appears larger and distant road recedes. The camera follows the selected driver and falls back to MAP if spatial resolution fails.

Canonical source data already preserves Z. Browser geometry can expose it, and the frontend enables elevation only when quality checks pass.

MAP and its original follow behavior remain intact.

## 14. Architecture comparison

| Concern | v1 | v2 |
| --- | --- | --- |
| Historical delivery | FastAPI + WebSocket | immutable static artifacts |
| Replay clock | server-paced frames | browser-owned timestamp clock |
| Main data artifact | application JSON | canonical Parquet + browser products |
| Race order | GPS-derived approximation | timing product |
| Telemetry delivery | embedded/limited | lazy detailed products |
| Storage | repo/server files | Cloudflare R2 |
| Publication | deployment/file copy | content-addressed release + atomic catalog |
| Failure handling | runtime behavior | validation + quarantine |
| Race discovery | hard-coded sessions | promoted catalog |
| Source acquisition | ad hoc | explicit residential boundary |
| Product UI | student full-stack app | motorsport product shell |
| Camera | top-down | MAP + additive CHASE |

## 15. What changed conceptually

The project moved from:

> Can I make F1 telemetry move smoothly on a screen?

to:

> Can I reconstruct historical races reproducibly, preserve data authority, validate difficult sessions, publish them atomically, serve them cheaply, and make the resulting product feel complete?

That is the core difference between v1 and v2.

## 16. Branch history

| Branch | Purpose |
| --- | --- |
| main | current v2 production source |
| github-releases-test | early release/distribution experiment |
| v2/milestone-1-static-replay | stateless replay proof |
| v2/milestone-2-data-foundation | canonical analytical layer |
| v2/milestone-3-delivery-benchmark | browser delivery contract |
| v2/milestone-4-race-intelligence | timing-authoritative race facts |
| v2/milestone-4b-edge-case-validation | hostile-race validation |
| v2/milestone-5-object-storage | immutable publication/R2 |
| v2/milestone-5c-library-scaling | catalog and archive scaling |
| v2/milestone-6-product-shell | product shell, visual redesign, archive growth, CHASE |
| archive/v1-original-deployment | preserved original v1 production state |
| archive/pre-circuit-intelligence-deployment | preserved v2 state before circuit-intelligence extension |

Historical milestone branches are preserved as frozen records rather than rewritten to match current documentation.

## 17. Current next steps

Near term:

- investigate quarantined races;
- add completed 2026 races through the normal publication path;
- tune CHASE camera smoothness without weakening the authoritative spatial model;
- refresh historical releases with official circuit metadata where useful;
- finish mobile, cross-browser, performance, and accessibility QA.

Future directions include DUEL, STRATEGY, spatial telemetry overlays, shareable race moments, and generated broadcast-style cameras.
