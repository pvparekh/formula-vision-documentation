# Formula Vision v2: Milestone 6: Product Shell + Visual System

**Status: accepted**

## Objective

Turn the proven replay/data platform into a product that feels designed for
race analysis rather than a developer demo.

Milestone 6 is not a cosmetic reskin. It establishes the reusable information
architecture and interaction system that Battle Mode, Strategy View, Driver
Focus and the event timeline will build on.

## Product principles

- race truth remains in canonical/pipeline products; presentation does not
  invent timing facts;
- the center of gravity is the race, not navigation chrome;
- dense information should be legible before it is decorative;
- motion communicates state changes and hierarchy, never hides latency;
- every core action remains keyboard accessible;
- the viewer must stay useful on laptop-sized screens before adding large
  desktop flourishes;
- loading, unavailable and degraded-data states are first-class product states;
- visual polish must not put hot 60 FPS telemetry through React rendering.

## Visual language

Formula Vision uses a restrained motorsport/broadcast system:

- near-black asphalt background rather than pure black everywhere;
- neutral elevated surfaces with thin low-contrast borders;
- Formula-red only for brand/action emphasis, not as generic decoration;
- monospaced numerals for timing, gaps, laps and telemetry;
- condensed uppercase micro-labels for broadcast-style metadata;
- large editorial typography on discovery surfaces;
- driver/team colors remain data colors and are not overridden by theme;
- subtle grid/technical texture is allowed on low-information surfaces;
- shadows are sparse and primarily separate floating overlays.

The design should feel closer to a modern timing/analysis workstation than a
dashboard template.

## Phase 6A: Scalable race explorer

Requirements:

- catalog-driven race count and year filters;
- no hard-coded season list or race count;
- search by race, year, round or location;
- latest-published race shortcut;
- useful loading, empty and catalog-failure states;
- layout that still works when the library grows from six races to hundreds.

## Phase 6B: Viewer shell

Recompose the active race into stable regions:

    global navigation / race identity
    session intelligence strip
    timing tower | track workspace | contextual driver panel
    race control / event overlays
    timeline + playback controls

The track workspace should receive the largest visual area. Timing and driver
detail are supporting analytical panes, not equal-sized cards.

## Phase 6C: Broadcast timing hierarchy

Timing tower rows should make the following readable at a glance:

- position and position change;
- driver code/team color;
- interval ahead and gap to leader;
- tyre compound / age;
- pit state;
- last lap;
- fastest-lap marker;
- retirement/classification state;
- selected/followed driver state.

Timing truth comes from the canonical timing/intelligence products.

## Phase 6D: Driver focus shell

The selected-driver pane should have stable regions for:

- driver identity and classification;
- speed / gear / throttle / brake / DRS;
- current/best lap and sectors;
- stint/compound/tyre age;
- position/gap context;
- eventual comparison and lap traces.

The shell should not assume every telemetry channel is available. Degraded
source data must render explicitly rather than collapse the layout.

## Phase 6E: Timeline foundation

Playback should evolve from a bare range input into a race timeline that can
render semantic markers without owning race truth.

The timeline API should be ready for:

- pit in/out;
- fastest lap changes;
- SC / VSC / red / yellow state changes;
- race-control messages;
- eventual battle/overtake highlights.

Marker clicks seek the authoritative replay clock.

## Responsive behavior

Desktop target: timing tower + track + driver panel in one analytical canvas.

Smaller laptop: driver detail may overlay the track and secondary metadata can
collapse.

Tablet/mobile is not expected to show the entire workstation simultaneously;
it should provide an intentional track-first stacked experience rather than
shrinking desktop panels until they are unreadable.

## Performance gates

- no per-frame React state for track positions;
- no unbounded replay or telemetry buffers;
- no duplicate race-manifest fetches;
- immutable data remains cacheable;
- product-shell animation honors prefers-reduced-motion;
- frontend production build remains green after each phase.

## Exit criteria

Milestone 6 is accepted when:

- race explorer works with arbitrary catalog years/race counts;
- viewer shell has a coherent visual hierarchy on desktop and laptop widths;
- timing tower, track workspace, driver focus and playback share one visual
  system;
- loading/error/degraded states are explicit;
- keyboard controls remain intact;
- no regression is introduced in canonical replay semantics;
- production frontend build is green;
- representative races from the hostile-race matrix are visually smoke-tested.

Milestone 7 then adds the signature analysis workflows on top of these stable
surfaces.

## Acceptance record

The complete accepted six-race publication fixture was exercised locally through
the same catalog -> immutable manifest -> immutable artifact resolver used by
the production R2 path:

- 2023 Dutch Grand Prix;
- 2024 Bahrain Grand Prix;
- 2024 Monaco Grand Prix;
- 2024 Canadian Grand Prix;
- 2024 São Paulo Grand Prix;
- 2024 Qatar Grand Prix.

All six opened through the catalog-driven race explorer and canonical replay
path. The M6 branch's frontend production build and publication-contract test
suite are green.

Accepted product surfaces include:

- scalable race explorer and race switching;
- session intelligence strip;
- canonical broadcast timing tower;
- track-first replay workspace;
- canonical Driver Focus with explicit unavailable telemetry states;
- semantic event timeline with authoritative-clock seeking;
- race-control overlay;
- responsive results presentation;
- honest pre-race/loading/error states;
- keyboard replay/focus controls;
- local development mounting of the accepted immutable publication graph.

Post-acceptance visual work remains non-blocking for race correctness. The
shared circuit-intelligence system now provides numbered/named turn context in
MAP and CHASE. Remaining visual work is primarily camera smoothness,
cross-browser tuning, accessibility, and future higher-order analysis modes.


## Post-acceptance extensions

Milestone 6 remained the product-shell boundary, but several product-level extensions were implemented after formal acceptance.

### Telemetry Cinema visual system

The race archive and viewer were redesigned around a more product-specific motorsport visual language:

- Barlow Condensed for display/race typography;
- Instrument Sans for interface text;
- IBM Plex Mono for timing/data;
- season shelves instead of a uniform card grid;
- real circuit geometry in race cards;
- perspective hover motion for track previews;
- contextual floating carousel controls;
- cinematic race briefing;
- quieter viewer chrome;
- integrated transport dock;
- refined timing rail and driver inspector;
- redesigned results and non-happy states.

The detailed design direction is documented in [UI_REDESIGN_BLUEPRINT.md](UI_REDESIGN_BLUEPRINT.md).

### Archive scaling

The catalog-driven archive was expanded beyond the original hostile-race fixture.

By the 2025 publication run, the promoted catalog contained 85 races. Twenty-two 2025 races were published and two remained quarantined for later investigation.

The library now behaves like a growing historical product rather than a six-race validation demo.

### Additive CHASE camera

The stable MAP renderer and existing top-down driver follow remain the default.

A second opt-in CHASE mode was added with:

- circuit-relative track progress;
- local tangent, normal, and curvature;
- continuity-aware driver projection;
- filtered source-derived lateral vehicle position;
- a presentation-only ideal guide that does not drive the car;
- perspective-correct track ribbon and restrained racing groove;
- contextual upcoming-turn callouts from shared circuit intelligence;
- bounded camera response to motion and corner geometry;
- safe fallback to MAP when spatial state cannot be resolved;
- optional quality-gated elevation support.

See [SPATIAL_CAMERA_ARCHITECTURE.md](SPATIAL_CAMERA_ARCHITECTURE.md).

CHASE is considered functional but still requires visual debugging and tuning before any default-camera decision.
