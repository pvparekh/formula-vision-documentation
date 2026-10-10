# Formula Vision v2: R2 Live Proof Runbook

**Historical acceptance runbook.** Use [PRODUCTION_RUNBOOK.md](PRODUCTION_RUNBOOK.md) for the deployed operating path. This file preserves the original Phase 5B live-proof procedure.

This runbook is for Milestone 5B only. The bucket should remain private during
the storage-contract proof. Public delivery/custom-domain work belongs to
Phase 5C.

## Cloudflare setup

1. Enable R2 for the Cloudflare account.
2. Create one **Standard** storage bucket, for example
   `formula-vision-v2`.
3. Keep public access disabled for the Phase 5B proof.
4. Create an **Account R2 API token** with **Object Read & Write** permission,
   scoped only to this bucket.
5. Record the Account ID, Access Key ID and Secret Access Key securely. The
   secret is only shown once.

Do not paste credentials into chat, source control, issue comments or logs.

## Local environment

PowerShell:

```powershell
$env:R2_ACCOUNT_ID="<account-id>"
$env:R2_ACCESS_KEY_ID="<access-key-id>"
$env:R2_SECRET_ACCESS_KEY="<secret-access-key>"
$env:R2_BUCKET="formula-vision-v2"
$env:R2_KEY_PREFIX="staging"
```

Leave `R2_PUBLIC_BASE_URL` unset for Phase 5B. If the bucket uses a
jurisdiction-specific S3 endpoint, set `R2_ENDPOINT_URL` to the exact endpoint
shown by Cloudflare.

Install the pinned SDK dependency after pulling the M5 branch:

```powershell
python -m pip install -r pipeline/v2/requirements.txt
```

Secret-safe configuration check:

```powershell
python pipeline/v2/publish_library.py --check-r2-config
```

Authenticated bucket check:

```powershell
python pipeline/v2/publish_library.py --store r2 --check-r2-connection
```

## Live publication proof

Start from a one-race catalog:

```powershell
python pipeline/v2/publish_library.py `
  --store r2 `
  --web-root artifacts/validation/edge-cases/web `
  --race 2024:1
```

Save the returned one-race `catalog_id`.

Expand the same staging catalog to the complete accepted six-race set:

```powershell
python pipeline/v2/publish_library.py `
  --store r2 `
  --web-root artifacts/validation/edge-cases/web `
  --all
```

Independent verification:

```powershell
python pipeline/v2/publish_library.py `
  --store r2 `
  --verify-only
```

Rollback to the saved one-race catalog:

```powershell
python pipeline/v2/publish_library.py `
  --store r2 `
  --rollback-catalog "<one-race-catalog-id>"
```

Verify that the promoted catalog contains one race, then republish `--all`.
The six-race catalog id must return to the exact previously observed value and
all already-uploaded immutable race objects must be reused.

## Phase 5B acceptance

Phase 5B is accepted only when:

- bucket authentication succeeds;
- one-race publication verifies;
- incremental expansion to six races verifies;
- all R2 object metadata passes the cache contract;
- rollback verifies the old graph before promotion;
- rollback produces a one-race verified catalog;
- republishing all six restores the exact prior six-race catalog identity;
- the final repeated six-race publish creates zero immutable objects;
- no credentials appear in repository history or command output.

Only after this proof do we configure public delivery, CORS and a custom domain
for Phase 5C.
