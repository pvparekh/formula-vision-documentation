# Formula Vision v2: Data Source, Attribution and Publication Policy

> Engineering policy, not legal advice.

## Purpose

Formula Vision separates the software license of a source library from the rights and terms that may apply to upstream motorsport data.

FastF1 is an unofficial open-source Python package licensed under MIT. That software license does not automatically grant Formula Vision permission to redistribute every upstream dataset that FastF1 can access.

## Current publication rule

Until an explicit source/terms review is completed:

- canonical Parquet snapshots are development/build artifacts and are not published as a bulk public dataset;
- raw FastF1 responses and caches are not committed to the public repository;
- public browser payloads should contain only the derived data required to power Formula Vision;
- near-live/public redistribution is treated as a separate higher-risk feature and is not part of the historical replay rollout;
- source/version/transform provenance is recorded in build manifests;
- Formula Vision must make clear that it is unofficial and is not associated with Formula 1, the FIA or any team;
- Formula 1 marks/logos/visual identity must not be used in a way that implies official affiliation.

## Architecture consequence

This is another reason to separate layers:

FastF1/source cache -> internal canonical artifacts -> validated product-specific web artifacts -> browser

The canonical layer exists for reproducibility, quality assurance, analysis and regeneration. The public delivery layer exists to serve the product and can intentionally expose a smaller surface.

## Before large-scale public backfill

Complete a documented review of:

- the then-current Formula 1 digital product terms/guidelines;
- FastF1's current notices/documentation;
- the exact fields and degree of transformation in the proposed public artifacts;
- attribution/disclaimer requirements;
- whether any alternative source is preferable for schedule/results/classification data;
- live-mode authentication and redistribution implications.

Publication automation must be easy to disable independently of extraction/validation so the engineering pipeline can still be developed safely.