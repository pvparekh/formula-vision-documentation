# ADR 003: Decouple upstream source acquisition from hosted CI

**Status:** Accepted

## Context

During Milestone 2, GitHub-hosted Actions could load basic session
results/schedule metadata but could not acquire Bahrain 2024 timing, lap,
telemetry, weather or race-control channels from the Formula 1 timing service.

This matches a broader 2026 behavior where industrial/datacenter IPs can be
rejected while residential networks still have access.

A deterministic build pipeline should not depend on a network path that is known
to be environment-sensitive.

## Decision

Treat source acquisition as its own boundary.

- Hosted CI runs deterministic contract/unit/schema tests.
- Full FastF1 acquisition runs from an environment/network that can legitimately
  access the source, or from an already populated FastF1 cache.
- Transformation, validation and artifact generation remain deterministic after
  source acquisition.
- A failed acquisition publishes nothing.
- The extractor produces an explicit source-acquisition error instead of
  continuing with partial session data.
- Future automation may use a self-hosted/approved acquisition worker or another
  compliant source strategy, but the analytical pipeline does not depend on the
  acquisition transport.

## Consequences

Positive:

- CI is reliable and reproducible;
- network/provider behavior is not confused with transformation correctness;
- source caches can be managed independently;
- we avoid brittle proxy workarounds;
- pipeline failure modes become clearer.

Tradeoffs:

- fully unattended cloud ingestion is not available through ordinary hosted
  runners today;
- race acquisition needs an approved runtime/cache;
- new-race automation is constrained by upstream access and terms.

## Rejected alternative

Retrying hosted runners, hiding acquisition failures as warnings, or adding
proxy circumvention would make the pipeline less predictable and could create
additional operational or terms-of-use risk.
