# Formula Vision v2: Cloudflare Public Delivery

This runbook describes the public read path for the Formula Vision R2 race
library after the private publication contract has passed.

It intentionally does **not** require sharing R2 access credentials with the
frontend. Browser delivery is public read-only HTTP through Cloudflare; the S3
API credentials remain private publication credentials.

## Target shape

    Formula Vision frontend
            |
            | HTTPS GET
            v
    public R2 origin
    r2.dev or custom domain
            |
            v
    existing R2 bucket
            |
            v
    staging/catalog/latest.json
    staging/catalogs/<catalog-id>.json
    staging/races/<...>/<release-id>/...

The frontend base URL therefore points at the object prefix, not at the S3 API
endpoint:

    VITE_RACE_LIBRARY_BASE_URL=https://<public-race-library-origin>/staging

## Security boundary

Never put these values into Vite/Vercel public environment variables:

- R2_ACCESS_KEY_ID
- R2_SECRET_ACCESS_KEY
- any R2 API token

They are publisher credentials only.

The browser needs one non-secret value:

- VITE_RACE_LIBRARY_BASE_URL

The Cloudflare S3-compatible endpoint is also not the browser delivery URL.

## Cloudflare handoff

Use the existing Formula Vision R2 bucket that passed Phase 5B. The
Cloudflare-managed `r2.dev` URL is sufficient for the direct public read
contract. Attach a custom domain only when branding or explicit edge-cache
configuration is required.

The bucket CORS policy can be configured with the same private R2 S3
credentials already used by the publisher; no browser secret is involved:

    python -m pipeline.v2.configure_r2_cors \
      --origin https://formulavision.vercel.app \
      --origin http://localhost:5173

Inspect it later with:

    python -m pipeline.v2.configure_r2_cors --check

If a custom domain is desired, attach it in the Cloudflare dashboard. Wrangler
also supports this operation for an authenticated local Cloudflare session:

    npx wrangler r2 bucket domain add <bucket> \
      --domain <data-domain> \
      --zone-id <zone-id>

Do not paste Cloudflare/R2 credentials into source control or frontend
configuration.

Keep the existing staging object prefix during acceptance. Do not copy the
objects into a second bucket merely to make them public.

Configure CORS narrowly for the origins that actually need browser access:

- the Formula Vision production origin;
- a Vercel preview origin only if preview testing is required;
- localhost development only if local browser testing is required.

Required browser methods are GET and HEAD. Formula Vision does not write R2
objects from the browser and does not require cross-origin credentials.

Cloudflare only returns CORS response headers when the request includes an
`Origin` header. After changing CORS on a custom domain that has already served
traffic, purge that hostname's cache so old cached responses without the new
CORS headers are not retained.

## Cache contract

Publication already writes the intended **browser/origin metadata**:

Immutable race objects and immutable catalogs:

    Cache-Control: public, max-age=31536000, immutable

Mutable pointer:

    Cache-Control: no-store, max-age=0

Cloudflare configuration must preserve this distinction.

Important: R2 custom domains do not cache every file type by default. Formula
Vision publishes JSON and MessagePack artifacts, so production should use an
explicit Cloudflare Cache Rule for the immutable path families rather than
assuming those responses will automatically enter edge cache.

Recommended rules:

1. **Cache immutable Formula Vision objects**

       /staging/races/*
       /staging/catalogs/*

   Configure these paths as eligible for cache ("Cache Everything" / equivalent
   Cache Rule behavior) while respecting the long origin Browser TTL.

2. **Bypass the mutable pointer**

       /staging/catalog/latest.json

   Explicitly bypass edge cache for this path. Its origin metadata remains:

       Cache-Control: no-store, max-age=0

Do not add one blanket long-lived cache rule over the entire staging prefix.
That could cache `catalog/latest.json` and delay newly promoted races.

The immutable paths are content-addressed, so long-lived caching is safe: a
changed race or catalog gets a different URL instead of mutating the old URL.

Cloudflare also recommends Smart Tiered Cache for R2 custom-domain delivery to
reduce repeated R2 origin reads across edge locations. It is optional for the
first launch but is a clean later optimization.

## Frontend configuration

After the chosen public R2 origin is serving the objects, configure the
frontend deployment:

    VITE_RACE_LIBRARY_BASE_URL=https://<public-race-library-origin>/staging

Then rebuild/redeploy the frontend. Vite environment variables are compile-time
inputs, so changing the deployment environment without a rebuild is not enough.

No R2 credentials are required by the frontend build.

## Acceptance sequence

Use an already-published staging catalog. Do not perform another historical
upload only to test browser delivery.

1. Fetch the pointer through the configured public R2 origin.
2. Confirm it resolves to the accepted six-race immutable catalog.
3. Fetch the immutable catalog through the public origin.
4. Open Formula Vision with VITE_RACE_LIBRARY_BASE_URL configured.
5. Confirm the landing page lists the six catalog races.
6. Open Bahrain 2024 and one hostile-race fixture.
7. Confirm canonical positions, timing, intelligence, telemetry and results
   load through the immutable release paths.
8. Seek using the semantic timeline and select at least one driver.
9. Refresh and repeat a race load to exercise edge/browser caching.
10. Confirm the mutable pointer response is not cached as immutable content.

The browser resolver is intentionally fail-closed. A broken catalog hash,
unsafe path, identity mismatch or missing artifact inside an already-resolved
immutable release should display an error rather than silently load unrelated
legacy race data.

## Header checks

The exact command-line HTTP client is not important. The acceptance evidence
should prove:

Pointer:

    Content-Type: application/json
    Cache-Control: no-store, max-age=0

Immutable catalog/race objects:

    Cache-Control: public, max-age=31536000, immutable

CORS should return an Access-Control-Allow-Origin value compatible with the
actual Formula Vision browser origin.

## Cutover discipline

Do not remove the local schema-v3/schema-v2/WebSocket migration fallbacks
during this first public-delivery proof.

First prove the R2/CDN resolver in a browser against representative races.
Fallback removal is a separate cutover decision after the historical library
and production domain are stable.

## Cost behavior

Public historical replay remains static-first:

- no always-on database;
- no always-on historical API server;
- one tiny mutable catalog pointer;
- immutable race files cache at the edge/browser;
- repeated playback reads reuse the same content-addressed assets;
- new races require compute/publication work once, not per viewer.

The production deployment should continue to use Cloudflare budget alerts and
the library backfill byte/object guardrails. Provider pricing must be checked
against current Cloudflare documentation rather than hard-coded into the
application.

## Phase 5C accepted when

- public R2 delivery is reachable from the deployed browser;
- CORS is limited to intended browser origins;
- pointer and immutable cache semantics are preserved;
- VITE_RACE_LIBRARY_BASE_URL is configured without any secret;
- six-race catalog discovery works end-to-end;
- representative replay/intelligence/telemetry/results loads work;
- browser errors remain fail-closed;
- no legacy backend is required for the accepted canonical races.
