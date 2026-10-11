# Formula Vision v2 Documentation

This directory is the documentation home for my Formula Vision project and rewrite.

As of October 2026, the promoted historical library contains **170+ races**.


## Start here

- [Engineering Overview](Documentation%20%28FULL%29/ENGINEERING_OVERVIEW.md): current product, architecture, pipeline, replay engine, operations, and engineering scope in one system-level document.
- [Architecture](Documentation%20%28FULL%29/ARCHITECTURE.md): current system boundaries, data flow, publication model, browser runtime, and failure model.
- [Project Evolution](Documentation%20%28FULL%29/PROJECT_EVOLUTION.md): how the college-era v1 became the current data and software engineering system.
- [Roadmap](Documentation%20%28FULL%29/ROADMAP.md): accepted milestones, current state, and future work.

## Operations and how-to guides

- [Production Runbook](Documentation%20%28FULL%29/PRODUCTION_RUNBOOK.md)
- [Historical Library Backfill](Documentation%20%28FULL%29/LIBRARY_BACKFILL.md)
- [Cloudflare Delivery](Documentation%20%28FULL%29/CLOUDFLARE_DELIVERY.md)
- [Cutover Checklist](Documentation%20%28FULL%29/CUTOVER_CHECKLIST.md)
- [R2 Setup](Documentation%20%28FULL%29/R2_SETUP.md)

## Data and validation reference

- [Data Source Policy](Documentation%20%28FULL%29/DATA_SOURCE_POLICY.md)
- [Data Attribution](Documentation%20%28FULL%29/DATA_ATTRIBUTION.md)
- [Bahrain 2024 Canonical Profile](Documentation%20%28FULL%29/BAHRAIN_2024_CANONICAL_PROFILE.md)
- [Delivery Benchmark Findings](Documentation%20%28FULL%29/DELIVERY_BENCHMARK_FINDINGS.md)

## Product and interaction design

- [Milestone 6 Product Shell](Documentation%20%28FULL%29/MILESTONE_6.md)
- [UI Redesign Blueprint](Documentation%20%28FULL%29/UI_REDESIGN_BLUEPRINT.md)
- [UI References](Documentation%20%28FULL%29/UI_REFERENCES.md)
- [Spatial Camera Architecture](Documentation%20%28FULL%29/SPATIAL_CAMERA_ARCHITECTURE.md)

## Architecture decisions

- [ADR 001: Static historical delivery](Documentation%20%28FULL%29/001-static-historical-delivery.md)
- [ADR 002: Canonical data vs browser artifacts](Documentation%20%28FULL%29/002-canonical-data-vs-web-artifacts.md)
- [ADR 003: Source acquisition boundary](Documentation%20%28FULL%29/003-source-acquisition-boundary.md)
- [ADR 004: Browser delivery contract](Documentation%20%28FULL%29/004-browser-delivery-contract.md)

## Milestone history

1. [Milestone 1: Stateless historical replay](Documentation%20%28FULL%29/MILESTONE_1.md)
2. [Milestone 2: Canonical data foundation](Documentation%20%28FULL%29/MILESTONE_2.md)
3. [Milestone 3: Browser delivery benchmark](Documentation%20%28FULL%29/MILESTONE_3.md)
4. [Milestone 4: Race intelligence](Documentation%20%28FULL%29/MILESTONE_4.md)
5. [Milestone 4B: Hostile-race validation](Documentation%20%28FULL%29/MILESTONE_4B.md)
6. [Milestone 5: Object storage and library automation](Documentation%20%28FULL%29/MILESTONE_5.md)
7. [Milestone 6: Product shell](Documentation%20%28FULL%29/MILESTONE_6.md)

## Documentation conventions

- Keep the root README focused on what the project is, why it matters, how to run it, and where to read more.
- Put operational procedures in runbooks.
- Put current technical truth in the engineering overview, architecture, and reference docs.
- Put major tradeoff decisions in ADRs.
- Preserve milestone docs as historical records.
- Update documentation in the same change that modifies architecture or operating procedure.
- Prefer relative repository links.
- Keep current-state documentation timeless where possible; reserve date-relative language for milestone/history records.
- Prefer concrete constraints, measurements, and decisions over self-evaluative claims.
- Use direct, concise language and consistent terminology.
- Do not use em dashes in repository text.

These conventions follow GitHub README guidance, the Diataxis documentation model, and Google's developer documentation style principles.
