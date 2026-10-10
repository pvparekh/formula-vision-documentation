# Formula Vision v2: Production Runbook

This runbook operates the deployed v2 replay platform: frontend deployment,
race ingestion, immutable publication, delivery verification, bounded storage
cost, rollback, and recurring maintenance. Historical replay does not require
an always-on application backend.

## Production architecture

    FastF1
       |
       v
    canonical build + validation
       |
       v
    browser artifact build + validation
       |
       v
    publication preflight
       |  actual R2 prefix bytes
       |  candidate byte/object limits
       |  9.0 GB storage budget
       v
    immutable R2 release objects
       |
       v
    immutable race catalog
       |
       v
    catalog/latest.json (CAS promoted last)
       |
       v
    public Cloudflare R2 delivery origin
       |
       v
    Vercel Formula Vision frontend

Historical replay remains static-first. No database or always-on replay API is
required for canonical races.

## Current production state

- the v2 React/Vite frontend is deployed on Vercel;
- the race-library base URL is a Vite compile-time environment variable and is
  configured for both preview and production builds;
- historical race data is delivered separately from the frontend through the
  public Cloudflare R2 origin;
- `main` is the current v2 production source;
- `archive/v1-original-deployment` preserves the original v1 production state;
- `archive/pre-circuit-intelligence-deployment` preserves the v2 state before
  the circuit-intelligence extension.

A frontend rollback and a race-catalog rollback are separate operations. Do not
conflate application source with the promoted historical data graph.

## Source-acquisition execution model

A real 2024 full-season acceptance was attempted on a GitHub-hosted Ubuntu
runner. All 24 races failed at the acquisition boundary before canonical
publication: 22 because the F1 timing endpoints refused the cloud/datacenter
source path and the final two after FastF1 reached its 500 calls/hour hard
limit.

That result is useful: **full-fidelity acquisition must not be scheduled on a
generic cloud runner.** Formula Vision now keeps acquisition on a
residential-network machine where the six hostile fixtures were successfully
built.

There are two supported residential paths:

1. run `scripts/update_race_library.ps1` directly (the simplest option);
2. register the Windows machine as a GitHub self-hosted runner with the custom
   label `formula-vision-ingest`, then let GitHub Actions orchestrate the same
   updater.

An optional Windows Scheduled Task installer is included:

    .\scripts\install_weekly_update_task.ps1

The task uses `StartWhenAvailable`, so a missed Monday run can start the next
time the machine is available.

The FastF1 cache is persistent between local runs. If the upstream 500
calls/hour guard is reached during a large historical bootstrap, Formula Vision
stops making new acquisition requests and reports remaining races as
`DEFERRED_RATE_LIMIT`. The `scripts/backfill_history.ps1` bootstrap waits 65
minutes and resumes automatically by default; `--missing-only` skips races
already promoted. Pass `-NoAutoResumeRateLimit` to opt out of the wait.

## GitHub configuration

The workflow is:

    .github/workflows/publish-race-library.yml

### Required Actions secrets

Configure these in GitHub Actions Secrets. Do not commit them and do not place
them in Vite/Vercel public environment variables.

    R2_ACCOUNT_ID
    R2_ACCESS_KEY_ID
    R2_SECRET_ACCESS_KEY
    R2_BUCKET

The R2 token should be scoped only as broadly as publication requires.

### Repository variables

Recommended:

    R2_KEY_PREFIX=staging
    FV_PUBLIC_BACKFILL_APPROVED=false

The scheduled current-season publisher remains skipped while
`FV_PUBLIC_BACKFILL_APPROVED` is not `true`. See
`docs/v2/DATA_ATTRIBUTION.md`.

### Concurrency

All production publication runs share one GitHub Actions concurrency group:

    formula-vision-r2-publisher

Only one publisher is allowed to mutate the catalog at a time. The underlying
catalog compare-and-swap remains the final safety boundary.

## Storage budget

Automated publication uses a 9.0 decimal-GB store budget.

Preflight measures the **actual bytes already stored under the configured R2
prefix**, then conservatively adds the candidate payload plus a small
catalog/pointer reserve.

Behavior:

- well below the ceiling: publish normally;
- near the ceiling: report a warning;
- projected above 9.0 GB: block before mutation;
- manual workflow dispatch can explicitly override the budget.

The projection intentionally over-counts candidate objects that may already
exist. This makes the default safe and simple. If a retry near the ceiling is
blocked conservatively, use a dry-run/report first and override only when the
stored bytes and intended publication are understood.

The budget is a Formula Vision safety rail, not a guarantee that an entire
Cloudflare account can never incur charges.

## Current R2 economics

As of October 2026, Cloudflare documents a Standard R2 free tier of 10 GB-month
of storage per month, 1 million Class A operations and 10 million Class B
operations, with Internet egress free.

Current pricing reference:

https://developers.cloudflare.com/r2/pricing/

Provider pricing can change. Formula Vision therefore encodes a conservative
storage ceiling but does not hard-code provider prices into application logic.

Recommended account-level safety:

- enable Cloudflare billing notifications/alerts;
- review the R2 usage page after large historical backfills;
- re-check current R2 pricing periodically.

## Manual publication workflow

Use Actions -> **Formula Vision - Publish Race Library**.

The workflow supports:

- current-season;
- one full season;
- one race.

Manual runs default to **dry-run**.

For the first full-season acceptance:

1. choose target `season`;
2. enter `2024`;
3. leave dry-run enabled;
4. inspect the generated report and storage projection;
5. rerun with dry-run disabled;
6. verify the final catalog;
7. rerun the same season with the normal missing-only behavior and confirm
   already-published identities are skipped.

Per-race failures remain quarantined and are not promoted.

## Automatic future races

After the public-data gate is approved, set:

    FV_PUBLIC_BACKFILL_APPROVED=true

The scheduled workflow runs weekly on Monday UTC **on the labeled self-hosted
residential runner**. It discovers completed races in the current season,
compares them with the promoted catalog and uses `--missing-only`.

If nothing is missing, it exits without rebuilding old races.

If a completed race is missing:

    acquire -> canonicalize -> validate -> build web -> validate
            -> cost preflight -> publish -> verify

A race that fails validation is quarantined instead of poisoning the catalog.

## Cloudflare public delivery

Follow:

    docs/v2/CLOUDFLARE_DELIVERY.md

Use the public R2 delivery origin for the existing bucket/prefix. A custom
delivery domain can be attached later without changing race identities or the
frontend contract.

Preserve:

Immutable objects:

    Cache-Control: public, max-age=31536000, immutable

Mutable pointer:

    Cache-Control: no-store, max-age=0

Because Formula Vision serves JSON/MessagePack artifacts that Cloudflare does
not necessarily cache by default, add an explicit Cache Rule that makes these
content-addressed path families cache eligible:

    /staging/races/*
    /staging/catalogs/*

and explicitly bypass:

    /staging/catalog/latest.json

CORS should allow the deployed Formula Vision origin and only the development
origins actually needed. Purge the custom-domain cache after changing CORS on a
hostname that has already served traffic.

Do not expose R2 API credentials to the browser.

## Frontend production configuration

Set `VITE_RACE_LIBRARY_BASE_URL` for every Vercel environment that should use
the published race library, including preview builds used for release testing:

    VITE_RACE_LIBRARY_BASE_URL=https://<public-race-library-origin>/staging

Redeploy after changing the value because Vite environment variables are
compile-time inputs. A build created without this variable can fall back to the
small repository fixture catalog and must not be promoted as the production
v2 application.

Production canonical races will then resolve:

    catalog/latest.json
      -> immutable catalog
      -> immutable race manifest
      -> positions/timing/laps/intelligence/telemetry/results

No frontend code change is required when new catalog races are published.

## Production smoke test

For every production release, verify through the real public delivery origin:

- race library loads from the promoted catalog;
- Bahrain plus multiple hostile-race fixtures open;
- timing tower advances correctly;
- track replay advances and seeks;
- semantic timeline markers seek the authoritative clock;
- Driver Focus loads timing and available telemetry;
- missing telemetry is explicit rather than fabricated;
- race-control events appear;
- results load;
- switching races works without a frontend redeploy;
- reload/cached loads work;
- catalog pointer is not cached as immutable content.

## Rollback

Race releases and catalogs are immutable.

Rollback means promoting a previously verified catalog id. Do not delete
release objects during an incident.

The publisher verifies the target graph before changing
`catalog/latest.json`.

## Post-launch maintenance

Do not build a complex billing subsystem preemptively.

Near-term maintenance is intentionally boring:

- GitHub publication reports;
- Cloudflare usage/billing alerts;
- periodic pricing review;
- fix quarantined races deliberately;
- later add a conservative unreferenced-release garbage collector only if
  superseded immutable releases become meaningful storage overhead.

## Full operational maturity definition

The deployed baseline is already live. Treat the following as the remaining
operational maturity gates rather than as prerequisites for serving v2:

- M6 frontend build and pipeline CI are green;
- public custom-domain R2 delivery passes the browser smoke test;
- Vercel uses `VITE_RACE_LIBRARY_BASE_URL`;
- one full season passes the backfill/publish/missing-only cycle;
- GitHub Actions can add a future completed race without a React edit;
- the 9 GB pre-mutation budget is active;
- rollback is documented and previously proven;
- data attribution/publication gate is explicitly decided.
