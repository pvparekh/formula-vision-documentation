# Formula Vision: Data Attribution and Public Backfill Gate

Formula Vision is an unofficial fan/engineering project. It is not affiliated
with, endorsed by, or sponsored by Formula 1, the FIA, any team, FastF1, or
their affiliates.

## Software dependency

FastF1 is open-source software distributed under the MIT License. Formula
Vision uses FastF1 as an ingestion/analysis library and records the FastF1
version in canonical lineage metadata.

FastF1's software license does **not** by itself grant redistribution rights to
all upstream race data that FastF1 can access.

References:

- https://github.com/theOehrly/Fast-F1
- https://docs.fastf1.dev/

## Upstream data

Formula Vision's canonical datasets may include timing, lap, tyre/stint,
weather, race-control, position and car-telemetry fields obtained through
FastF1 from upstream services.

Those upstream services can have their own terms, trademarks and data-use
restrictions. The fact that data is technically retrievable is not a legal
conclusion that every derived raw artifact may be republished at arbitrary
scale.

The publication pipeline therefore separates:

- private/local extraction and validation;
- private R2 staging;
- public delivery.

## Public bulk-backfill gate

Scheduled public publishing is disabled unless the repository variable below is
set deliberately:

    FV_PUBLIC_BACKFILL_APPROVED=true

Keep it false until the intended public distribution model, attribution and
current upstream terms have been reviewed.

This gate is intentionally operational rather than embedded into race data. It
can be changed without rebuilding the pipeline.

Manual workflow dispatch remains available for deliberate staging/proof runs.

## Product attribution

The public product should retain a visible Data Source link to FastF1 and an
unofficial-project disclaimer.

Recommended copy:

> Unofficial motorsport replay and analysis project. Race data is processed
> with FastF1. Formula 1 and related marks belong to their respective owners.

Avoid implying an official Formula 1 partnership or endorsement.

## Source lineage

For reproducibility and future audits, canonical release metadata should retain:

- event/session identity;
- extraction/build timestamp;
- FastF1 version;
- canonical pipeline version;
- browser artifact pipeline version;
- source quality/warnings;
- artifact hashes and row counts.

## Review cadence

Because upstream terms and endpoints can change, re-check this gate before a
large new public backfill or if FastF1/F1 changes its access model.

This document is an engineering release gate, not legal advice.
