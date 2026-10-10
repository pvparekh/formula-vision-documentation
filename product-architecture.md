# Formula Vision v3 product architecture

## Product thesis

Formula Vision should feel like a focused motorsport product, not a dashboard that
exposes every capability at once.

The primary interaction path is:

```text
ARCHIVE -> SEASON -> RACE BRIEFING -> REPLAY
```

Each step answers one question:

1. **Archive** — What season do I want to explore?
2. **Season** — What race do I want to enter?
3. **Race briefing** — Is this the race I want, and am I ready to start?
4. **Replay** — What is happening in this race?

The interface reveals only the controls required to answer the current question.
This keeps the product cinematic without sacrificing discoverability.

## Why this hierarchy

The previous landing page exposed the product as a conventional content website:
large marketing hero, latest-race feature, search, year filters, then multiple
season shelves. The race cards and race briefing were strong individually, but
the page hierarchy asked users to process too many levels at once.

V3 makes the archive itself an interaction.

The structure follows the same broad principles seen in mature game and platform
interfaces:

- progressive disclosure rather than displaying the entire hierarchy at once;
- a small number of stable top-level destinations;
- contextual actions kept inside the mode they operate on;
- predictable focus and a persistent path back;
- global search as a second way to locate content in a large collection;
- motion used to preserve context between states, not as decoration.

## Information architecture

### 1. Archive

The Archive is the product's primary content gateway.

It owns:

- season selection;
- archive scope and counts;
- global race discovery;
- entry into a season.

It does **not** expose race-analysis tools, driver telemetry, playback settings,
or results.

The default Archive state is season-first. A user sees one highlighted season
and one representative race visual at a time. The season rail is the navigation
surface, not a filter attached to a giant page.

### 2. Season

A season is a focused catalog mode.

It owns:

- the season title and race count;
- race cards for that season;
- quick switching between available seasons;
- a clear path back to Archive;
- entry into Race Briefing.

Existing race cards remain the primary race-selection component because they
already encode circuit geometry, identity, metadata, motion, and interaction
well.

The Season view deliberately removes:

- the marketing hero;
- the sticky search/filter dashboard;
- simultaneous shelves for every other season;
- redundant information already represented by race cards.

### 3. Race Briefing

Race Briefing is the transition from browsing to an active session.

It owns:

- race identity;
- location;
- lap/driver/duration summary;
- canonical track geometry;
- start action.

The briefing should remain visually isolated and cinematic. It should not become
a menu of analysis features.

### 4. Replay

Replay is a dedicated working mode.

It owns every control that requires an active race:

- playback;
- timing tower;
- race-control events;
- map/chase camera;
- driver selection and telemetry;
- driver detail;
- lap/current-state context;
- results.

These are not separate top-level products. Their meaning depends on the selected
race and current replay state, so they remain contextual inside Replay.

## Persistent product shell

The 52 px top bar is the stable cross-mode shell.

### Left

- Formula Vision identity.
- In Replay, a direct Archive/back action.

### Center

- Archive: archive scope/status.
- Replay: active race identity and round.

### Right

- Global race search / race switcher.
- Project information utility.

Search is deliberately persistent. A large archive should have more than one
way to locate a race: hierarchical browsing and direct search.

Project links and data-source attribution are utilities, not peer navigation
destinations.

## Global command search

Keyboard shortcut: `/`

Search covers race name, circuit/location, year, and round from one place.

Keyboard contract:

- typing refines results immediately;
- Up/Down changes the active result;
- Enter opens the active race;
- Escape closes search;
- pointer hover and keyboard selection share the same visible highlight.

Search can bypass the Archive -> Season browsing path intentionally. It is the
fast path for users who already know what they want.

## Archive keyboard contract

At the season-selection level:

- Left/Right changes the highlighted season;
- Enter opens the focused season;
- `/` opens global search.

Inside a season:

- Escape returns to season selection;
- race cards remain standard focusable buttons;
- horizontal scrolling remains available through mouse, trackpad, touch, and
  contextual rail arrows.

The currently selected season is preserved when a user enters a race and later
returns to Archive.

## Navigation state

Archive state is intentionally remembered by the application:

```text
Archive
  -> 2025
      -> Las Vegas
          -> Replay
          -> Back
      -> returns to 2025
```

Returning from a race should not reset the user's browsing context.

## Motion system

Motion communicates hierarchy and continuity.

### State transitions

- Archive -> Season: directional transition into a deeper level.
- Season -> Archive: directional transition back toward the parent.
- Race card -> Briefing: existing race-entry behavior remains.
- Briefing -> Replay: existing focused transition remains.

### Rules

- direct manipulation should respond immediately;
- large state transitions should be short and decisive;
- decorative ambient motion must never compete with selection focus;
- use transforms and opacity for frequently animated UI;
- avoid animating layout or expensive filters on pointer hot paths;
- reduced-motion mode must remain visually complete.

A strong still frame is a prerequisite. Animation enhances hierarchy; it does
not rescue weak composition.

## Visual system

### Hierarchy

The visual hierarchy is:

1. current decision;
2. selected content;
3. contextual metadata;
4. utility controls;
5. decorative atmosphere.

The interface should never make utility chrome visually compete with the race or
season being selected.

### Materials

Use material separation sparingly:

- near-black base;
- restrained translucent raised surfaces;
- thin neutral borders;
- race accent color as environmental light and state feedback;
- white for decisive actions/focus;
- muted neutral typography for metadata.

Avoid turning every region into a glass card.

### Spatial composition

Full-screen atmosphere may span the page, but content panels must have clear
semantic boundaries. Large ambient fields should not reveal their raster or
container edges.

The strongest existing components — race cards and Race Briefing — are visual
anchors for the rest of the system.

## Feature-boundary rules

Do not create top-level destinations simply because a feature can be imagined.

### Drivers

A top-level Drivers destination becomes valid only when Formula Vision supports
meaningful cross-race or cross-season driver workflows such as:

- season performance comparison;
- telemetry comparison across races;
- driver history;
- head-to-head analysis.

Driver information for the currently active race remains inside Replay.

### Data / Analysis Lab

A top-level analysis destination becomes valid only when users can perform
independent analytical workflows that do not require entering a replay first,
for example:

- querying race/lap/telemetry datasets;
- comparing races;
- exporting derived analysis;
- building custom visualizations.

Raw data-source attribution is not enough to justify a top-level Data section.

### Sandbox

A Sandbox becomes valid only when the product has a true exploratory or
simulation capability, for example:

- what-if race scenarios;
- configurable telemetry overlays;
- custom replay composition;
- lap/driver comparative experiments;
- strategy simulation.

Until then, adding a Sandbox tab would create information architecture without a
user job behind it.

### Settings

Settings should be introduced only for persistent user preferences. Contextual
replay controls such as camera mode and playback speed remain inside Replay.

Likely future persistent settings:

- reduced visual motion beyond OS preference;
- default camera;
- telemetry density;
- preferred units;
- accessibility/display options.

## Performance contract

The shell must remain lightweight relative to replay rendering.

- Keep the archive SVG-based.
- Do not introduce WebGL solely for menu decoration.
- Lazy-load replay-only heavy components.
- Prefer transform/opacity animation.
- Use `will-change` only around active interaction.
- Avoid continuously repainting large blurred regions.
- Keep global search client-side over the already-loaded catalog.
- Preserve reduced-motion fallbacks.
- Do not duplicate race data to support navigation.

## Accessibility contract

- Every primary destination has a predictable path back.
- Keyboard navigation follows visual direction.
- Focus appearance is visible on interactive controls.
- Search is usable without a pointer.
- Functional state is never communicated by accent color alone.
- Decorative visuals remain non-semantic.
- Responsive reflow must not create two-axis page scrolling.
- Reduced-motion users receive the same hierarchy and functionality.

## Current v3 implementation

Implemented on `v3-product-shell-archive`:

- season-first Archive;
- highlighted season preview;
- keyboard-aware season rail;
- focused Season catalog;
- preservation of the existing race cards;
- persistent archive context across race entry/return;
- universal race command search;
- keyboard search selection;
- compact project/data-source utility;
- existing Race Briefing and Replay preserved;
- responsive and reduced-motion states.

## Next component passes

Future visual work should proceed from the inside out:

1. replay shell and information density;
2. timing tower hierarchy;
3. driver detail hierarchy;
4. playback/control language;
5. race-control events;
6. results;
7. empty/loading/error states;
8. optional settings after real preferences exist.

Each pass should preserve this product hierarchy rather than creating a new
navigation model.

## Reference principles

The architecture was informed by a broad review of game UI, platform design,
accessibility, and web-performance guidance, including:

- Apple Human Interface Guidelines: Layout, Menus, Materials, Motion, Search,
  Focus and Selection, Navigation and Search;
- Microsoft Xbox Accessibility Guidelines: UI Navigation, UI Context, Input,
  Focus Handling, Visual Distractions and Motion;
- W3C WAI-ARIA Authoring Practices: listbox/grid/tabs keyboard interaction;
- WCAG 2.2 focus, target, and motion guidance;
- MDN guidance for View Transitions, animation performance, transforms,
  compositing, containment, and `will-change`;
- Call of Duty's documented move from a cluttered HQ model to fewer,
  self-contained hubs with direct access;
- Forza / Sea of Thieves examples used by Xbox for predictable game-menu
  navigation;
- high-end interactive work from Active Theory and Lusion, particularly their
  emphasis on simple navigation, focused content, state-driven experiences, and
  performance discipline.

Primary public references:

- https://developer.apple.com/design/human-interface-guidelines/layout
- https://developer.apple.com/design/human-interface-guidelines/menus
- https://developer.apple.com/design/human-interface-guidelines/materials
- https://developer.apple.com/design/human-interface-guidelines/motion
- https://developer.apple.com/design/human-interface-guidelines/searching
- https://developer.apple.com/design/human-interface-guidelines/search-fields
- https://developer.apple.com/design/human-interface-guidelines/focus-and-selection/
- https://learn.microsoft.com/en-us/gaming/accessibility/xbox-accessibility-guidelines/112
- https://www.w3.org/WAI/ARIA/apg/patterns/listbox/
- https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API
- https://www.callofduty.com/blog/2024/10/call-of-duty-black-ops-6-road-to-launch-preload-new-ui-intel
- https://www.callofduty.com/blog/2024/08/call-of-duty-optimizing-experience-roadmap
