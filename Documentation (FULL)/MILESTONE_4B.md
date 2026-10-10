# Formula Vision v2: Milestone 4B: Hostile-Race Validation

**Status: accepted**

## Objective

Prove that the accepted Bahrain race-intelligence architecture survives races
that violate Bahrain's assumptions before any whole-season backfill or object-
storage publication work begins.

A session that cannot satisfy its validation contract is quarantined. It is
never silently promoted into the future public race catalog.

## Validation matrix

| Case | Purpose |
| --- | --- |
| Bahrain 2024 | Accepted dry/normal control |
| Canada 2024 | Wet/dry transitions, multiple Safety Cars, restarts, DNFs |
| Netherlands 2023 | Heavy rain, VSC, red flag, restart, high pit volume |
| Monaco 2024 | Lap-1 red flag, restart, immediate DNFs, unusual tyre strategy |
| São Paulo 2024 | Wet race, VSC, Safety Car, red flag, DNS/disqualification-style classification |
| Qatar 2024 | Sprint-weekend event metadata, multiple Safety Cars, DNFs and lapped classification |

The expected characteristics are encoded in `pipeline/v2/edge_case_matrix.py`.
They are assertions, not documentation-only labels.

## Runner

The fail-closed runner is:

```bash
python pipeline/v2/validate_edge_cases.py --case canada-2024-wet-safety-car --acquire-missing
```

Run the complete matrix with:

```bash
python pipeline/v2/validate_edge_cases.py --all --acquire-missing
```

Acquisition is local only. Hosted CI does not attempt FastF1 source acquisition
because upstream F1 endpoints have already proven unreliable from datacenter
runners.

For each case the runner:

1. reuses an existing canonical snapshot or acquires it locally when explicitly
   requested;
2. builds disposable M4 web artifacts under
   `artifacts/validation/edge-cases/web`;
3. reruns the production web-artifact hash/contract validator;
4. asserts the expected hostile-race characteristics;
5. emits PASS, QUARANTINED or MISSING_CANONICAL;
6. writes machine-readable JSON and human-readable Markdown reports.

One bad race does not stop the rest of the matrix.

## Generic release gates

Every passing case must have:

- correct session identity;
- M4-or-newer browser artifacts;
- matching manifest/intelligence driver sets;
- a canonical fastest lap;
- at least 95% coverage in every published telemetry channel;
- valid artifact hashes and row/column contracts.

Race-specific gates then verify facts such as rain, Safety Car, VSC, red flag,
early red flag, pit-event volume, non-finishers, zero-lap classifications,
lapped classifications and non-conventional sprint-weekend event format.

## Quarantine semantics

`QUARANTINED` is a successful safety mechanism, not an instruction to weaken
the assertion.

When a case is quarantined:

1. identify whether the failure is source acquisition, canonical modeling,
   browser artifact generation, or a race-specific expectation;
2. inspect the canonical source and official race facts;
3. fix the data contract or explicitly document a source limitation;
4. add a regression test;
5. rerun the quarantined case;
6. only then mark it PASS.

## Exit criteria

Milestone 4B is complete when:

- Bahrain remains PASS;
- Canada PASS;
- Netherlands PASS;
- Monaco PASS;
- São Paulo PASS;
- Qatar PASS;
- any discovered edge-case fixes have regression tests;
- rerunning the full matrix is idempotent;
- the report contains zero QUARANTINED cases.

Only then does work move to Milestone 5 object storage, atomic publication and
full-library automation.


## Driver availability and lifecycle contract

Hostile sessions force a distinction that the Bahrain baseline did not need:

- all result/session drivers remain in metadata, results and race intelligence;
- only drivers with usable position samples are spatial replay drivers;
- only drivers with usable car data receive lazy telemetry artifacts;
- missing position/car telemetry is acceptable only for an explicit non-starter
  or withdrawn driver; missing replay data for a driver who raced quarantines
  the session;
- FastF1 alignment padding is not browser telemetry. Rows without source
  provenance are excluded from the delivery layer while remaining preserved in
  canonical Parquet;
- long spatial gaps are held rather than interpolated through, which prevents
  cars from flying across the circuit during red-flag/feed pauses;
- a retired/disqualified driver's live timing state expires after their timing
  feed ends, while official classification/status remains available;
- the timing tower and Driver Focus present DNS/DSQ/DNF/no-data states instead
  of inventing an ON TRACK state.

The web manifest therefore exposes explicit spatial-driver and telemetry-driver
subsets plus missing-driver quality metadata.


## Accepted hostile cases

### Canada 2024: PASS

Canada is the first hostile-race acceptance case.

Observed production metrics:

- 20 drivers;
- 1,238 non-deleted timed laps;
- zero official timing exclusions;
- 27 rainfall samples;
- 20 track-status transitions;
- 128 race-control events;
- 87 pit events;
- 5 nonfinishers;
- 1 classified lapped driver;
- no missing position or car-telemetry drivers;
- motion-aware release telemetry minimum: 99.6813% (SAR gear);
- raw telemetry minimum: 34.7337% (SAR throttle), preserved as diagnostic
  provenance rather than used as the release-quality denominator.

Canada exposed and validated the distinction between raw telemetry coverage and
motion-aware release coverage. FastF1 can continue emitting real car-source
rows after retirement while throttle is unavailable. The web contract now
preserves raw coverage for diagnostics, gates dynamic channels over moving
samples, and expires replay telemetry/map state after the final meaningful
motion sample.


### Netherlands 2023: PASS

The Dutch Grand Prix is the second hostile-race acceptance case and validates
the replay contracts through a wet race with VSC, red flag, restart and very
high pit activity.

Observed production metrics:

- 20 drivers;
- 1,309 non-deleted timed laps;
- zero official timing exclusions;
- 48 rainfall samples;
- 11 track-status transitions;
- 62 race-control events;
- 203 pit events;
- 3 nonfinishers;
- no missing position or car-telemetry drivers;
- motion-aware release telemetry minimum: 99.9957% (GAS throttle);
- raw telemetry minimum: 15.8181% (SAR throttle), preserved as diagnostic
  provenance;
- earliest red-flag event: 5,290,666 ms into the replay timeline.

This case confirms that long interrupted-session periods do not require
weakening telemetry quality gates. Motion-aware coverage remains near-complete
while stationary/post-running source degradation remains visible separately.
It also validates that the replay can carry weather, VSC/red-flag state,
restart-era timing and high-volume pit events through one immutable race
artifact set.


### Monaco 2024: PASS

Monaco is the third hostile-race acceptance case and validates the immediate
lap-one red-flag/restart path plus unusual classification structure.

Observed production metrics:

- 20 drivers;
- 1,221 non-deleted timed laps;
- zero official timing exclusions;
- earliest red flag at 56,773 ms;
- 11 track-status transitions;
- 109 race-control events;
- 46 pit events;
- 4 nonfinishers;
- 4 zero-lap classifications;
- 9 classified lapped drivers;
- no missing position or car-telemetry drivers;
- motion-aware release telemetry minimum: 99.8615% (SAI gear);
- raw telemetry minimum: 0.3215% (PER throttle), preserved as diagnostic
  provenance.

This case confirms that an immediate red-flag interruption does not require
weakening timing or telemetry contracts. The replay artifact set can preserve
classification, lapped-car state, zero-lap retirements and high-quality moving
telemetry while retaining severe stationary/post-running raw-source degradation
for diagnostics.


### São Paulo 2024: PASS

São Paulo is the fourth hostile-race acceptance case and validates the sparse
driver/data-availability contract against a real sprint-weekend race with wet
conditions, VSC/Safety Car/red-flag interruptions, DNS drivers and a
disqualification.

Observed production metrics:

- 20 official/session drivers;
- 1,076 non-deleted timed laps;
- zero official timing exclusions;
- 97 rainfall samples;
- 28 track-status transitions;
- 93 race-control events;
- 70 pit events;
- 5 nonfinishers;
- 3 zero-lap classifications;
- 1 disqualified driver;
- event format: sprint_qualifying;
- missing car telemetry: ALB and STR;
- nonstarter drivers: ALB and STR;
- zero unexpected missing position drivers;
- zero unexpected missing telemetry drivers;
- motion-aware release telemetry minimum: 99.9276% (BEA gear);
- raw telemetry minimum: 37.2030% (COL throttle), preserved as diagnostic
  provenance;
- earliest red flag at 2,844,484 ms.

This case confirms the web contract's core sparse-driver invariant in production:
official/session drivers are not assumed to equal spatial or car-telemetry
drivers. Genuine nonstarters remain present in classification/intelligence
without fabricated telemetry, while missing replay data for drivers who raced
continues to fail closed.


### Qatar 2024: PASS

Qatar is the fifth and final hostile-race acceptance case. It validates the
race contract on a sprint-weekend event with repeated Safety Car periods,
retirements, zero-lap classifications, lapped classification and high pit
activity.

Observed production metrics:

- 20 drivers;
- 905 non-deleted timed laps;
- zero official timing exclusions;
- 23 track-status transitions;
- 99 race-control events;
- 121 pit events;
- 5 nonfinishers;
- 2 zero-lap classifications;
- 1 classified lapped driver;
- event format: sprint_qualifying;
- no missing position or car-telemetry drivers;
- motion-aware release telemetry minimum: 100%;
- raw telemetry minimum: 1.19% (COL throttle), preserved as diagnostic
  provenance.

Qatar completes the hostile-case matrix. Every individual case now passes the
production builder, artifact validator and race-specific expectation layer.

## Final M4B gate

Before Milestone 4B is frozen, rerun the complete matrix in one process from
the already-acquired canonical snapshots:

```bash
python pipeline/v2/validate_edge_cases.py --all
```

This final gate proves batch idempotency and verifies that all six cases
(Bahrain control plus the five hostile races) still produce zero quarantined
sessions under one branch head. No source acquisition is required.


## Final acceptance record

Milestone 4B is accepted.

The complete matrix was rerun in one process from the acquired canonical
snapshots and produced:

- PASS: 6;
- QUARANTINED: 0;
- MISSING_CANONICAL: 0.

Passing sessions:

- Bahrain 2024 baseline;
- Canada 2024;
- Netherlands 2023;
- Monaco 2024;
- São Paulo 2024;
- Qatar 2024.

This satisfies the exit criteria: all individual hostile cases pass, discovered
edge-case fixes have regression coverage, the full matrix is idempotent under
one branch head, and no race is quarantined.

The accepted contract now distinguishes official/session drivers from spatial
and car-telemetry availability; excludes FastF1 alignment padding from browser
delivery; preserves raw telemetry diagnostics; gates replay-quality telemetry
over meaningful moving samples; expires retired-driver replay state; preserves
DNS/DSQ/classification semantics; and fails closed when a driver who raced is
missing required replay data.

Milestone 5 may begin from this checkpoint. Historical publication must retain
the validate-before-publish and quarantine semantics proven here.
