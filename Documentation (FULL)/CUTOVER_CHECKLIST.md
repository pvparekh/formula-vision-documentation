# Formula Vision v2: Cutover Checklist

**Status: completed historical procedure**

This file preserves the v1-to-v2 production cutover sequence. It is useful as
migration evidence and rollback context, but it is no longer the primary
deployment guide. Use [../../DEPLOYMENT.md](../../DEPLOYMENT.md) for the current
entry point and [PRODUCTION_RUNBOOK.md](PRODUCTION_RUNBOOK.md) for ongoing
operations.

## 1. Pull the cutover branch

On the residential Windows machine:

    cd C:\Dev\formula1raceviewer
    git fetch origin
    git switch v2/milestone-6-product-shell
    git pull --ff-only origin v2/milestone-6-product-shell

## 2. Build the historical library

The production bootstrap is resumable and safe to rerun:

    .\scripts\backfill_history.ps1 -StartYear 2018

It uses the existing private R2 credentials from the local environment and:

- discovers completed races only;
- skips already-published identities;
- acquires full-fidelity FastF1 source data on the residential network;
- builds and validates canonical data;
- builds and validates browser artifacts;
- quarantines bad races instead of publishing them;
- measures actual R2 prefix usage;
- blocks before mutation above the 9.0 GB storage budget;
- publishes immutable releases and atomically promotes the catalog.

If FastF1's 500 calls/hour limiter is reached, the bootstrap preserves the
cache and already-published races, waits 65 minutes, and resumes the same
season automatically. Pass `-NoAutoResumeRateLimit` only if you prefer to stop
and resume manually.

Do not use `-OverrideStorageBudget` during the initial historical bootstrap.

## 3. Attach the public R2 delivery domain

Choose a subdomain in a Cloudflare-managed zone, for example:

    data.example.com

Then run:

    .\scripts\connect_r2_public_domain.ps1 \
      -Domain "data.example.com" \
      -ZoneId "<cloudflare-zone-id>"

Wrangler will use the locally authenticated Cloudflare account. The script also
applies read-only browser CORS for the Formula Vision production origin.

Cloudflare may take a few minutes to mark the custom domain Active.

### One dashboard cache rule

R2 custom domains do not cache JSON/MessagePack by file type automatically.
Create one Cache Rule for the immutable object families:

Expression concept:

    Hostname equals data.example.com
    AND (
      URI Path starts with /staging/races/
      OR URI Path starts with /staging/catalogs/
    )

Settings:

- Cache eligibility: **Eligible for cache**
- Edge TTL: **Use cache-control header if present, bypass cache if not**

Do **not** include:

    /staging/catalog/latest.json

The pointer is intentionally `no-store`.

## 4. Verify Cloudflare delivery

After the custom domain is Active:

    .\scripts\finalize_public_delivery.ps1 \
      -PublicBaseUrl "https://data.example.com/staging" \
      -AppOrigin "https://formulavision.vercel.app" \
      -AllManifests

This checks the mutable pointer policy, immutable catalog hash, every promoted
manifest hash, CORS and immutable cache headers.

## 5. Point Vercel at the race library

Set the Formula Vision production environment variable:

    VITE_RACE_LIBRARY_BASE_URL=https://data.example.com/staging

Then redeploy. Vite reads the value at build time.

No R2 secret belongs in Vercel.

## 6. Production browser smoke test

On the deployed Formula Vision site:

- race explorer shows the promoted R2 catalog;
- open Bahrain plus at least two hostile fixtures;
- switch directly between races;
- play/pause/seek;
- click semantic timeline markers;
- inspect a driver;
- verify race-control overlays;
- open results;
- refresh and reopen a race.

A canonical publication failure must fail closed rather than silently mixing
legacy data.

## 7. Merge / cut over (completed) (completed)

Only after the real public R2 path passes:

- open/finish the v2 cutover PR;
- merge `v2/milestone-6-product-shell` to `main`;
- let the existing Vercel Git integration build production;
- confirm `https://formulavision.vercel.app` is serving v2.

## 8. Future races

The low-friction manual update is:

    .\scripts\update_race_library.ps1 -Year 2026

It discovers and publishes only missing completed races.

Optional automation is already included through:

- `.github/workflows/publish-race-library.yml` on a labeled residential
  self-hosted GitHub runner; or
- `.\scripts\install_weekly_update_task.ps1` as a Windows Scheduled Task.

Neither is required for launch. Manual weekly updates are sufficient and keep
the full-fidelity source boundary simple.
