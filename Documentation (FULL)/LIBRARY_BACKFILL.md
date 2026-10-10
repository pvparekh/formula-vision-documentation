# Formula Vision v2: Historical Library Backfill Runbook

This runbook covers Milestone 5D private/local library growth. It does **not**
remove the public-data attribution/rights gate documented in MILESTONE_5.md.

The backfill runner is:

    python -m pipeline.v2.library_backfill

Its pipeline is deliberately fail-closed:

    discover
      -> acquire canonical source when explicitly allowed
      -> canonical validation
      -> browser artifact build
      -> browser validation
      -> immutable release planning
      -> one batch publication
      -> one catalog promotion
      -> independent full-graph verification

A race that fails any pre-publication stage is quarantined in the report and
is omitted from the publication batch. Unrelated valid races can still
publish.

## Why publication is batched

A season should not perform one mutable catalog promotion per race.

The runner prepares every valid immutable release first and then calls the
Milestone 5 publication core once. This keeps the expensive/mutable portion of
the operation small:

    N validated race releases
            |
            v
    immutable uploads/reuse
            |
            v
    one immutable catalog
            |
            v
    one CAS promotion of catalog/latest.json

This is cheaper, easier to reason about, and reduces catalog contention.

## Start with planning/reuse

Use already-acquired canonical snapshots without contacting FastF1:

    python -m pipeline.v2.library_backfill `
      --race 2024:1 `
      --race 2024:8 `
      --race 2024:9

Missing canonical races are reported as MISSING_CANONICAL. They are not
silently acquired.

To rebuild browser artifacts from existing canonical truth:

    python -m pipeline.v2.library_backfill `
      --race 2024:1 `
      --rebuild-web

## Discover a season

The season selector asks FastF1 for the event schedule and, by default, omits
future rounds:

    python -m pipeline.v2.library_backfill `
      --season 2024

Repeat --season for multiple seasons.

This command still does **not** acquire a missing race unless
--acquire-missing is supplied.

## Acquire missing canonical races

Source acquisition is explicit because it is the slowest and least reliable
stage and can be affected by upstream timing-endpoint access:

    python -m pipeline.v2.library_backfill `
      --season 2024 `
      --acquire-missing

FastF1 cache data is reused from .cache/fastf1 by default.

Do not use --rebuild-canonical casually. It intentionally reacquires/rebuilds
existing canonical snapshots and requires --acquire-missing.

### Refresh circuit metadata across the published archive

Circuit corner metadata is different from an ordinary browser-only rebuild.
Older canonical snapshots did not store FastF1 `get_circuit_info()` responses,
so adding official turn numbers/coordinates requires deliberate source
reacquisition plus a new immutable release.

The guarded PowerShell wrapper exposes one explicit mode for this enrichment.

Plan the completed 2018-2026 archive first:

    .\scripts\backfill_history.ps1 `
      -StartYear 2018 `
      -EndYear 2026 `
      -RefreshCircuitMetadata `
      -DryRun `
      -ContinueOnQuarantine

After reviewing every per-season report, run the same command without
`-DryRun` to publish the new immutable releases:

    .\scripts\backfill_history.ps1 `
      -StartYear 2018 `
      -EndYear 2026 `
      -RefreshCircuitMetadata `
      -ContinueOnQuarantine

`-RefreshCircuitMetadata` deliberately enables all three required behaviors:

- rebuild canonical snapshots so FastF1 circuit information can be captured;
- rebuild browser manifests with `metadata.circuit`;
- refresh already-published identities instead of applying the normal
  `--missing-only` skip.

The normal library updater remains missing-only by default. Published release
refresh is opt-in so routine updates cannot silently republish the archive.

If source circuit information is unavailable for one race, canonical extraction
records a warning and the race may still publish. The frontend then uses
geometry-derived turn inference for that release. Quarantine behavior and all
other validation guardrails remain unchanged.

The 2026 selector still follows the normal completed-round policy. Scheduled
future rounds are not fabricated or published before source race data exists.

## Dry-run the exact R2 publication

With the Phase 5B R2 environment variables set:

    python -m pipeline.v2.library_backfill `
      --season 2024 `
      --missing-only `
      --publish `
      --dry-run `
      --store r2

This builds/validates selected races and computes the exact immutable release
and catalog plan without writing R2 objects or promoting the pointer.

The generated report includes candidate race count, object count and raw bytes
so a large publication is visible before mutation.

## Publish missing races to private staging

During Milestone 5 keep the R2 key prefix isolated:

    $env:R2_KEY_PREFIX="staging"

Then:

    python -m pipeline.v2.library_backfill `
      --season 2024 `
      --acquire-missing `
      --missing-only `
      --publish `
      --store r2

--missing-only reads the currently promoted catalog and skips identities
already present there. Content-addressed immutable releases make retries safe
even if a prior attempt uploaded objects but failed before catalog promotion.

## Multi-season private backfill

After at least one full season has passed validation:

    python -m pipeline.v2.library_backfill `
      --season 2023 `
      --season 2024 `
      --acquire-missing `
      --missing-only `
      --publish `
      --store r2

Do not jump directly to every season before proving the first full-season
report. New edge cases should improve the validator/pipeline rather than be
worked around manually.

## Quarantine reports

Default report location:

    artifacts/validation/library-backfill/
      report.json
      report.md

Per-race statuses:

- PUBLISHED: validated and included in the promoted catalog;
- READY: validated and release-planned but not published (or dry-run);
- SKIPPED_PUBLISHED: omitted by --missing-only;
- MISSING_CANONICAL: no canonical snapshot and acquisition was not enabled;
- QUARANTINED: a build/validation/release-planning stage raised an error.

The JSON report includes the failed stage, exception type/message and bounded
traceback. Quarantined races are never passed to publish_releases.

## Bounded object-store concurrency

The publisher/backfill defaults to 8 workers and accepts 1–32:

    python -m pipeline.v2.library_backfill `
      --season 2024 `
      --workers 12

Concurrency is limited to immutable object I/O and verification. The mutable
catalog/latest.json promotion remains one serialized compare-and-swap write.

This is intentional: speed up independent work without weakening publication
ordering or concurrent-publisher safety.

## Mutation guardrails

Large backfills can enforce a hard pre-mutation object/byte ceiling in addition
to using `--dry-run`.

For example, refuse to mutate R2 if the selected validated candidates exceed
2 GiB or 5,000 immutable objects:

    python -m pipeline.v2.library_backfill `
      --season 2023 `
      --season 2024 `
      --missing-only `
      --publish `
      --store r2 `
      --max-publish-mib 2048 `
      --max-publish-objects 5000

The limits are evaluated after validation/release planning and **before any
publication write**. A violating real publication exits with a
`PublicationGuardrailError` and leaves the catalog untouched.

Dry-runs still compute the exact plan when a limit would be exceeded. The
report records `would_block_mutation` and the violations so operators can
adjust the selection or limit deliberately instead of guessing.

## FastF1 source-access and rate-limit behavior

Formula Vision deliberately fails closed when FastF1 cannot obtain laps/timing,
car telemetry and position telemetry.

GitHub-hosted/datacenter runners are not a reliable acquisition environment for
the current F1 timing endpoints. Use a residential/local machine or a
residential self-hosted GitHub runner for `--acquire-missing`.

For Windows, the repository includes:

    .\scripts\update_race_library.ps1 -Year 2024 -DryRun
    .\scripts\update_race_library.ps1 -Year 2024

The second command is guarded by `--missing-only`, the R2 mutation limits and
the store-level storage budget.

FastF1 also enforces a hard global request limit. If that limit is reached,
Formula Vision marks the current and remaining not-yet-attempted races as
`DEFERRED_RATE_LIMIT` and stops making source requests. Keep the cache and
rerun after the rate-limit window resets. Already-promoted race identities will
be skipped.

This is preferable to sleeping blindly inside CI or weakening source fidelity.

## Store-level storage budget

For production R2 publication, the orchestrator can protect the **actual object
store footprint**, not only the size of the current candidate batch:

    python -m pipeline.v2.library_backfill \
      --season 2024 \
      --acquire-missing \
      --missing-only \
      --publish \
      --store r2 \
      --storage-budget-gb 9.0

The preflight lists the configured R2 prefix and sums its real stored object
bytes. It then conservatively adds the candidate immutable payload plus a 16
MiB catalog/pointer reserve.

The 9.0 value is decimal GB so it can be compared directly with provider
GB-month storage reporting.

When projected storage exceeds the ceiling, a real publication fails before
mutation. Dry-runs still report the projection.

A deliberate manual publication can override only this store-level ceiling:

    --override-storage-budget

The override is reported explicitly. Scheduled Formula Vision publication never
uses it.

The projection intentionally does not subtract already-existing candidate
objects. That can overestimate a retry, but it keeps the default logic simple
and fail-safe. Use the report plus an explicit override only when the difference
is understood.

## Cost behavior

The design minimizes repeated cloud work:

- canonical acquisition is cached locally;
- existing canonical/web artifacts are reused unless rebuild flags are set;
- --missing-only skips catalog identities already published;
- published refresh is explicit and creates new immutable releases rather than
  mutating an existing release;
- immutable keys are content-addressed, so retries reuse identical objects;
- one backfill batch performs one catalog promotion;
- browser clients load the tiny no-store pointer, then immutable cacheable
  catalog/race objects;
- historical replay requires no always-on database or application server.

Do not hard-code provider pricing into the pipeline. The report exposes object
and byte counts so current Cloudflare pricing can be applied externally.

## Public-data gate

Private/local backfill validation may proceed.

Before bulk historical data is exposed through the production Formula Vision
domain, complete the FastF1/source attribution and Formula 1 data-rights review
already required by Milestone 5.
