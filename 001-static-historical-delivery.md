# ADR 001: Serve historical replay as immutable static data

**Status:** Accepted

## Context

Formula Vision v1 streamed precomputed historical race frames through a stateful
FastAPI/WebSocket service. The server loaded large JSON files into memory, owned
the playback clock and enforced a one-viewer limit to fit the original free-tier
memory budget.

Historical race data is immutable after it has been prepared.

## Decision

Historical v2 replay artifacts are delivered as versioned static objects. The
browser owns playback state and time, loads only required partitions, prefetches
ahead and keeps a bounded cache.

A backend is not required to pace historical frames.

## Consequences

Positive:

- no server cold start on the historical hot path;
- viewers do not share playback state;
- concurrency scales with static/CDN delivery;
- race data can be cached aggressively;
- idle compute cost approaches zero;
- seeking becomes direct random access to immutable artifacts.

Tradeoffs:

- the frontend needs a real playback runtime;
- artifacts require explicit versioning and validation;
- live sessions remain a separate architecture.

## Rejected alternative

Paying for a larger always-on Render instance would reduce symptoms while
preserving a server responsibility that the historical workload does not need.
