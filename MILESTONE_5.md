# Formula Vision v2: Milestone 5: Object Storage + Library Automation

**Engineering status: accepted**

## Objective

Turn race availability into a safe, idempotent publication operation.

Milestone 5 moves canonical browser artifacts out of the application bundle and
into object storage without reintroducing a stateful historical backend. Race
releases are immutable. One tiny catalog pointer is the only mutable
publication switch.

## Publication architecture

```text
FastF1/local cache
      |
canonical Parquet
      |
canonical validation
      |
browser artifact build
      |
web artifact validation
      |
immutable race release
      |
immutable catalog
      |
catalog/latest.json  <-- promoted LAST
      |
browser
```

The catalog points directly to an immutable race manifest. Race manifests keep
artifact paths relative to their own immutable release directory.

## Object layout

```text
races/<year>/<round>/<session>/<release-id>/
  manifest.json
  timing.json
  laps.json
  results.json
  intelligence.json
  positions/...
  telemetry/...
  track_status.json
  session_status.json
  race_control.json
  weather.json

catalogs/<catalog-id>.json
catalog/latest.json
```

`release-id` is content-addressed from every file path, byte count and SHA-256
inside the built race directory. Rebuilding identical bytes therefore produces
the same release path.

`catalog-id` is content-addressed from the complete ordered catalog.

## Atomic publication protocol

Publication is fail-closed:

1. validate every source race with the production web-artifact validator;
2. compute immutable race release identities without mutating storage;
3. load and verify the currently promoted catalog, if one exists;
4. upload immutable race objects;
5. read every written object back and verify exact bytes;
6. build and upload the immutable next catalog;
7. verify that catalog;
8. compare-and-swap `catalog/latest.json` LAST against the exact pointer
   version read during planning;
9. read the pointer back and independently verify the complete published graph.

The compare-and-swap requirement prevents concurrent publishers from silently
losing one another's catalog updates. If the pointer changed after planning,
promotion aborts and the publisher must re-plan from the new catalog. Uploaded
immutable objects remain safe and unreachable until a later catalog references
them.

A crash or upload failure before step 8 cannot expose a partial new release.
The previous pointer continues to reference a complete immutable catalog and
immutable race releases.

A stale client also remains safe: an old pointer references objects that are
never overwritten or deleted as part of normal publication.

A race release is **manifest-closed**: the publishable file set is exactly
`manifest.json` plus every artifact path referenced by the manifest. Missing
referenced files, duplicate artifact paths, traversal paths, symlinks, and
unexpected stray files all fail publication before storage mutation. Release
and catalog identities use the full SHA-256 digest rather than a shortened
display digest.

## Cache contract

Immutable race objects and immutable catalogs:

```text
Cache-Control: public, max-age=31536000, immutable
```

Mutable catalog pointer:

```text
Cache-Control: no-store, max-age=0
```

This matters when R2 is exposed through a cached custom domain: object-storage
origin consistency does not by itself prevent a CDN from serving an older
overwritten object.

## Phase 5A: Local publication proof

**Status: accepted**

No cloud account or credentials are required.

Core files:

- `pipeline/v2/publication.py`
- `pipeline/v2/publish_library.py`
- `pipeline/v2/tests/test_publication.py`

Required proofs:

- deterministic content-addressed release identity;
- independent post-publish verification;
- idempotent republish;
- incremental catalog update preserves existing races;
- immutable-key collision fails hard;
- injected failure before catalog promotion leaves the previous catalog live;
- concurrent/stale catalog promotion fails closed instead of overwriting a
  newer pointer;
- release directories are manifest-closed and reject stray/traversal files;
- published artifact tampering is detected.

Local example using the M4B artifact set:

```bash
python pipeline/v2/publish_library.py \
  --web-root artifacts/validation/edge-cases/web \
  --target-root artifacts/publication/local-store \
  --all
```

Independent verification:

```bash
python pipeline/v2/publish_library.py \
  --target-root artifacts/publication/local-store \
  --verify-only
```

Re-running the same publish must create zero new immutable objects and retain
the same catalog id.


### Phase 5A acceptance record

The accepted six-race M4B artifact set was published to the local object-store
implementation and independently verified.

First publication:

- races: 6;
- immutable objects created: 318;
- immutable objects reused: 0;
- independently verified objects: 319 (318 immutable objects plus the mutable
  catalog pointer);
- independently verified bytes: 156,175,013;
- catalog id: `sha256-ecff620c8ecdff7b6643e68aa72f4d9dbefee9c22949fc335d8c637cf49906c9`.

The same publication was then rerun without changing the source artifacts:

- immutable objects created: 0;
- immutable objects reused: 318;
- catalog id unchanged;
- pointer payload hash unchanged;
- independent verification remained green for all six races and 319 objects.

This proves deterministic content addressing, full-graph verification, and
idempotent replay of the complete accepted race set. Phase 5B may proceed.

## Phase 5B: R2 backend

**Status: accepted**

The S3-compatible R2 adapter reuses the exact same publication core. Cloud
specific behavior is limited to storage primitives: conditional PutObject,
opaque ETag version tokens, metadata verification, and bucket connectivity.

Expected configuration names:

- `R2_ACCOUNT_ID`;
- `R2_ACCESS_KEY_ID`;
- `R2_SECRET_ACCESS_KEY`;
- `R2_BUCKET`;
- `R2_PUBLIC_BASE_URL` (required before frontend/public-CDN cutover);
- `R2_ENDPOINT_URL` (optional override for jurisdiction-specific R2 endpoints).

Secrets must live in local environment configuration and GitHub Actions
Secrets, never source control.

### Phase 5B safety contract

- immutable objects use conditional create (`If-None-Match: *`) so an
  existing immutable key is never blindly overwritten;
- mutable catalog promotion uses the ETag returned by the same GetObject read
  as its `If-Match` compare-and-swap token;
- first pointer creation uses `If-None-Match: *`;
- a 412 precondition failure aborts promotion instead of retrying against stale
  state;
- Content-MD5 protects upload transport and SHA-256 remains Formula Vision's
  content identity;
- HeadObject verification checks Content-Type and Cache-Control after upload;
- immutable objects must report the one-year immutable cache policy;
- `catalog/latest.json` must report `no-store, max-age=0`;
- rollback verifies the complete target catalog graph before CAS-promoting its
  pointer;
- config inspection never prints access-key or secret-key values.

No R2 credentials are required by normal CI. The adapter is exercised through a
fake S3 client that models conditional-write/precondition behavior.

The public race-library origin can be the Cloudflare-managed `r2.dev` endpoint
or a custom domain attached to the bucket. A custom domain is preferable when
branding, cache rules, or additional edge controls justify it, but it is not a
data-contract requirement.

### Phase 5B acceptance record

The accepted M4B six-race artifact set was published to a real private
Cloudflare R2 bucket under the `staging/` prefix and exercised through the
complete live publication/rollback/idempotency sequence.

Connectivity and one-race proof:

- authenticated bucket connectivity: green;
- Bahrain 2024 release id:
  `sha256-d2fcf9538dcba00794a7d07f51f53dd737ccc618e3285c68be5f63af0811b475`;
- one-race catalog id:
  `sha256-19bb5f6b8348ce30ba98ee4911a6e74b4fc4609b4ea5038fbcad1dee94c0b9e5`;
- one-race verification: 50 objects / 22,501,601 bytes.

Six-race expansion:

- catalog id:
  `sha256-ecff620c8ecdff7b6643e68aa72f4d9dbefee9c22949fc335d8c637cf49906c9`;
- immutable objects created during incremental expansion: 270;
- immutable race objects reused from Bahrain: 48;
- verification: 319 objects / 156,175,013 bytes;
- the live R2 catalog id exactly matched the accepted local Phase 5A catalog
  id, proving deterministic publication identity across storage backends.

Rollback and restore:

- rollback to the saved one-race catalog verified 1 race / 50 objects;
- republishing all six restored the exact prior six-race catalog id;
- restore created 0 immutable objects and reused all 318 immutable objects;
- independent verification after restore remained green for all 6 races,
  319 objects, and 156,175,013 bytes.

Final unchanged six-race rerun:

- immutable objects created: 0;
- immutable objects reused: 318;
- catalog id unchanged;
- verification remained green for all 6 races, 319 objects, and
  156,175,013 bytes.

This closes the live R2 publication contract: conditional immutable writes,
atomic pointer promotion, deterministic identity, rollback, complete graph
verification, and unchanged-library idempotency all passed against real R2.

### Production automation and cost guardrails

The production branch now includes a guarded GitHub Actions publisher:

- manual current-season / season / single-race dispatch;
- weekly current-season discovery after race weekends;
- `--missing-only` so published race identities are not rebuilt;
- one global publisher concurrency group;
- secret-safe R2 configuration;
- JSON/Markdown backfill reports uploaded as workflow artifacts;
- candidate byte/object mutation limits;
- actual R2-prefix storage accounting;
- conservative 9.0 GB storage-budget preflight with explicit manual override;
- scheduled publication disabled until `FV_PUBLIC_BACKFILL_APPROVED=true`.

The storage budget uses actual bytes already present under the configured R2
prefix and blocks before publication mutation. It intentionally leaves headroom
below the provider's current Standard free storage allowance and does not rely
on provider prices inside application logic.

### Publication scale/performance hardening

After the live proof exposed network round-trip latency as the dominant cost,
the publication path was hardened for larger libraries without changing the
safety protocol:

- release planning can validate/hash independent races concurrently while
  preserving deterministic input order;
- immutable upload/reuse checks run with bounded workers (default 8, maximum
  32);
- full graph verification parallelizes independent manifests/artifacts;
- the R2 HTTP connection pool is sized for bounded concurrency;
- redundant HeadObject existence checks were removed where a single GetObject
  can establish both existence and exact bytes;
- compact progress events make long backfills observable;
- phase timings expose planning, publication and verification costs;
- catalog creation and `catalog/latest.json` compare-and-swap promotion remain
  serialized and unchanged.

Concurrency is intentionally limited to independent immutable work. The one
mutable publication switch is still promoted last with the exact version read
during planning.

## Phase 5C: Frontend catalog resolver

**Status: accepted**

The frontend stops constructing canonical URLs from
`/data/v3/<year>/<round>/R`.

Instead it:

1. fetches `catalog/latest.json`;
2. verifies the pointer contract;
3. fetches the immutable catalog;
4. resolves the selected race to its immutable manifest path;
5. resolves every race artifact relative to that manifest directory.

Legacy v2/WebSocket fallback remains available during migration until explicit
cutover.

Implementation notes:

- `frontend/src/data/raceLibrary.ts` owns the publication-discovery contract;
- `VITE_RACE_LIBRARY_BASE_URL` points at the public library root, including
  any key prefix such as `staging`;
- the mutable pointer is fetched with `cache: no-store`;
- immutable catalog and release objects use cache-friendly URLs;
- pointer, catalog, race identity and object paths are validated before use;
- catalog and manifest SHA-256 values are verified in browsers with Web Crypto
  when available;
- catalog/manifest requests are deduplicated in memory and the pointer has a
  short client refresh window;
- canonical replay, intelligence, telemetry and results resolve paths relative
  to the immutable manifest directory instead of reconstructing
  `/data/v3/<year>/<round>/R`;
- the landing page derives canonical race availability from the promoted
  catalog;
- legacy v2/WebSocket fallback remains isolated for migration compatibility.

The catalog resolver is deployed through the production Vercel frontend with
`VITE_RACE_LIBRARY_BASE_URL` pointing at the public R2 library. Preview builds
that exercise the full archive use the same compile-time variable.

## Phase 5D: Library/backfill automation

**Status: accepted**

`pipeline/v2/library_backfill.py` now supports:

- one or repeated explicit race/session identities;
- season discovery through the FastF1 event schedule;
- repeated seasons for multi-season runs;
- incremental `--missing-only` selection against the promoted catalog;
- explicit source acquisition with `--acquire-missing`;
- canonical/web reuse by default with explicit rebuild flags;
- publication dry-runs with object/byte estimates;
- bounded object-store concurrency;
- one batch publication/catalog promotion for every successful group;
- per-race quarantine with JSON and Markdown reports.

Every selected session follows:

```text
discover -> acquire when explicitly allowed -> canonicalize -> validate
         -> build web -> validate -> release plan -> batch publish -> verify
```

A failed session is quarantined with a machine-readable stage/error report and
is omitted from the publication batch. One failed race therefore does not block
unrelated successful races. The detailed operator workflow lives in
`docs/v2/LIBRARY_BACKFILL.md`.

The orchestration has been exercised beyond the original six-race fixture and
used to grow the promoted historical catalog to 85 races while preserving
per-race quarantine. Incremental missing-only publication remains the normal
update path.

## Publication safety and rollback

Normal publication never mutates an existing immutable race release or catalog.
The mutable pointer is never a blind last-write-wins update: promotion requires
an atomic conditional write matching the pointer observed during planning.

Rollback means promoting a previously verified catalog pointer. Old race
releases stay available, so rollback does not require copying or rebuilding
race files.

Garbage collection is a separate future maintenance operation and must never
run implicitly during publication.

## Public-data gate

Do not bulk-expose the historical race library publicly merely because the
pipeline can build it.

Before large public backfill:

- review FastF1/source attribution requirements and Formula 1 data-rights
  implications;
- document attribution in the product;
- confirm the intended public distribution model is acceptable.

Private/local build and publication validation can proceed before that review.

## R2-specific production assumptions

Cloudflare R2 provides an S3-compatible API. The browser can use a public
`r2.dev` origin or a custom domain. Custom domains are useful when additional
cache/branding controls are required; publication identity and browser
resolution do not depend on the hostname.

Bucket CORS must explicitly allow the deployed Formula Vision origin and local
development origins that are actually required.

## Exit criteria

Milestone 5 is complete when:

- Phase 5A local publication contract is green;
- R2 backend uses the same tested publication core;
- immutable object cache headers and no-store pointer headers are verified;
- frontend uses catalog resolution for canonical races;
- catalog publication is atomic and rollback-tested;
- at least the accepted M4B race set is staged and independently verified;
- one incremental republish changes exactly the intended race/catalog;
- one full batch rerun is idempotent;
- failed sessions are quarantined and omitted from publication;
- GitHub Actions can discover/build/publish missing races without putting
  credentials in source control;
- production publication is concurrency-limited and storage-budgeted before
  mutation;
- public bulk backfill remains gated on the source-rights/attribution review.
