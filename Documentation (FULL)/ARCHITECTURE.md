# Formula Vision v2 Architecture

## Purpose

This document describes the current Formula Vision v2 system.

Milestone documents explain how the architecture evolved. ADRs explain why major decisions were made. Runbooks explain how to operate the system. This file is the current-state technical map.

## System overview

    SOURCE / BUILD PLANE

    FastF1 + upstream timing sources
                  |
                  v
    residential acquisition environment
                  |
                  v
    canonical extraction
    Parquet + manifest + hashes + lineage
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
                  v
    quarantine or publish plan
                  |
                  v
    content-addressed immutable release
                  |
                  v
    Cloudflare R2


    DELIVERY / CLIENT PLANE

    Cloudflare R2
          |
          v
    public delivery origin
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
          + telemetry
          + events/results
          |
          v
    React browser runtime
          |
          + MAP
          + CHASE

## Source acquisition boundary

Source acquisition is separated from transformation and publication.

Generic hosted CI proved unreliable for full-season FastF1 acquisition because upstream timing endpoints can reject datacenter traffic and FastF1 can enforce a request-rate guard.

Full-fidelity acquisition therefore runs on a residential-network machine. GitHub Actions can orchestrate that machine through a self-hosted runner, but source acquisition remains a separate boundary.

A source-access failure is not treated as a pipeline-correctness failure.

## Canonical analytical layer

Canonical race data is typed, normalized, versioned, and stored as Parquet.

The snapshot preserves session identity, source lineage, drivers, results, laps, timing, weather, status/control, X/Y/Z position telemetry, detailed car telemetry, and optional circuit turn metadata.

Canonical artifacts are not shaped around one UI component. They are the durable rebuild layer.

## Race intelligence

Facts that should not be repeatedly inferred in the browser are derived once in the pipeline.

Examples include position/gap context, grid change, lap/sector facts, fastest-lap progression, stint/tyre context, pit windows, classification state, and semantic race events.

## Browser delivery layer

Browser products are generated from canonical truth for efficient access.

The delivery contract separates multi-driver position chunks, timing, laps, race intelligence, results, status/control/weather, lazy per-driver telemetry, track geometry, and optional circuit metadata.

Canonical Parquet is not the browser protocol.

## Timing authority

v1 inferred race order from GPS-derived distance. v2 does not.

| Question | Authority |
| --- | --- |
| Position | timing product |
| Gap to leader | timing product |
| Interval ahead | timing product |
| Official result | results product |
| Lap/sector fact | laps and race intelligence |
| Drawn spatial position | position telemetry |
| Speed/RPM/gear/throttle/brake/DRS | car telemetry |

The renderer may interpolate presentation. It does not become race authority.

## Browser replay runtime

The replay clock is client-owned.

The browser resolves the promoted catalog, verifies immutable identities, loads only the position chunks near the current replay time, prefetches nearby chunks, keeps a bounded cache, updates hot spatial state through refs and requestAnimationFrame, and loads selected-driver telemetry lazily.

Race timestamps define time.

## Publication model

Publication is immutable by default.

    validate
      -> plan
      -> read current pointer version
      -> publish/reuse immutable race objects
      -> exact read-back verification
      -> publish/reuse immutable catalog
      -> compare-and-swap catalog/latest.json
      -> verify promoted graph

The pointer is promoted last. A stale publisher cannot silently overwrite a newer catalog.

## Quarantine

One bad race must not corrupt the archive.

A race that fails acquisition, canonical validation, browser validation, or release planning can be quarantined with a machine-readable report.

Backfill tooling can continue publishing unrelated valid races when explicitly configured.

## Product shell

The race library provides season shelves, search/filter, real track previews, contextual carousel controls, and race briefing.

The race workspace provides top navigation, timing tower, race-control ribbon, track/spatial canvas, driver inspector, timeline, transport, and results.

## Camera architecture

### MAP

MAP is the default and proven renderer. It includes the whole-field top-down view and original driver-centered follow behavior.

### CHASE

CHASE is additive and opt-in.

It builds a circuit-relative spatial model containing closed circuit progress, tangent, lateral normal, curvature, and optional quality-gated elevation.

The implementation separates authoritative longitudinal progress, filtered source-derived lateral vehicle position, and a presentation-only ideal guide. The visible guide does not drive the car or define timing.

A followed driver is projected onto circuit progress with continuity constraints. The camera uses the road path and filtered vehicle state, while bounded motion response adds framing without redefining race truth.

CHASE also consumes the shared circuit-intelligence layer for contextual upcoming-turn callouts. It is lazy-loaded and falls back to MAP if spatial state cannot be resolved safely.

See [SPATIAL_CAMERA_ARCHITECTURE.md](SPATIAL_CAMERA_ARCHITECTURE.md).

## Circuit intelligence

Canonical extraction attempts to preserve FastF1 circuit markers such as turn number/letter, marker coordinates, angle, distance, and circuit rotation.

MAP and CHASE consume one shared resolver. Published circuit metadata is preferred when available. Older immutable releases can fall back to geometry-derived turn inference so presentation degrades gracefully without changing race authority.

Common corner names are curated display metadata layered on top of numbered turns. Unsupported names remain numbered turns rather than being invented.

## Elevation

Canonical position telemetry preserves X/Y/Z.

Browser track geometry can preserve optional Z. The frontend enables true elevation only when coverage and quality checks pass. Otherwise CHASE renders a flat circuit in perspective.

## Cache model

Immutable releases and catalogs:

    Cache-Control: public, max-age=31536000, immutable

Mutable latest pointer:

    Cache-Control: no-store, max-age=0

## Deployment topology

    Vercel production frontend
          |
          v
    public Cloudflare R2 delivery origin
          |
          v
    content-addressed race library

Cloudflare Pages can host the same frontend as an optional preview environment. Frontend hosting and race-data publication remain independent.

## Failure philosophy

Formula Vision prefers explicit failure over plausible-looking incorrect race data.

Examples:

- invalid race: quarantine;
- missing immutable artifact: fail the race;
- catalog pointer conflict: abort and re-plan;
- unusable CHASE geometry: return to MAP;
- missing telemetry channel: show unavailable state;
- source acquisition failure: report the source boundary instead of weakening validation.
