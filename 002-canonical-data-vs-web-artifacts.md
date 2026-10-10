# ADR 002: Separate canonical analytical data from browser delivery artifacts

**Status:** Accepted

## Context

The v1 frame JSON was simultaneously the pipeline output, backend storage model
and frontend protocol. Adding richer timing, strategy and telemetry data directly
to that structure would tightly couple every system and create very large web
payloads.

Different consumers have different access patterns.

## Decision

Formula Vision v2 has two data layers:

1. **Canonical analytical layer**: typed, normalized, versioned tables stored
   as Parquet for validation, reproducibility and offline analysis.
2. **Browser delivery layer**: product-specific partitions generated from the
   canonical layer and benchmarked for network size, decode cost, seek latency
   and browser memory.

The canonical schema is stable analytical truth. Browser serialization is an
optimization detail and may change independently.

## Consequences

Positive:

- FastF1-specific structures do not leak into React;
- web formats can be changed without rethinking analytical truth;
- validation and regression tests operate on clean tables;
- telemetry can be lazy-loaded by driver instead of duplicated into every frame;
- future consumers can reuse canonical data internally.

Tradeoffs:

- one additional transformation stage;
- schema/version management is required;
- canonical and delivery artifacts need independent validation.

## Rejected alternative

Expanding the existing frame JSON with every speed, tyre, sector, weather and
timing field would be simple initially but expensive to transfer, parse, evolve
and test.
