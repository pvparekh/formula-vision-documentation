# Formula Vision UI Redesign Blueprint

Status: design direction approved for exploration; implementation should proceed in phases and preserve the existing canonical replay/data contract.

## North star

Formula Vision should feel like a premium motorsport instrument, a modern media product, and a game menu at the same time.

The target is not "clean SaaS." It is **Telemetry Cinema**: a dark, precise, cinematic interface where the race itself is the artwork. The content layer carries motion, circuit geometry, driver color, timing, and race state. Navigation and controls stay restrained, fast, and predictable.

A person who knows nothing about the ingestion, validation, publication, timing authority, or telemetry pipeline should still immediately feel that the product is unusually polished and intentional.

The product should be recognizable from a cropped screenshot with the Formula Vision logo removed.

## What we are explicitly moving away from

The current v2 shell is strong structurally but visually still contains several patterns that now read as generic AI-produced web UI:

- Inter as the global typeface.
- Repeated rounded cards with similar borders, fills, shadows, and radii.
- Radial-gradient hero lighting as a default decorative device.
- Uniform three-column archive grids.
- Small uppercase, wide-tracked "kicker" copy used too often.
- Generic product-marketing phrases such as "Historical race intelligence."
- Repeated glassy/transparent rectangles used as hierarchy.
- Hover = translate upward + glow.
- Long explanatory copy in places where the interface should communicate through structure.
- Too many arbitrary Tailwind values instead of one deliberate visual grammar.

The redesign should not replace those defaults with a different trend pack. It needs a product-specific system.

## Design principles

### 1. The race is the hero

Circuit geometry, lap state, timing, telemetry, race control, tyres, and driver identity are the visual material. Decorative gradients should never carry more visual weight than actual race data.

### 2. Content is expressive; chrome is quiet

Apple's current design guidance treats controls/navigation as a distinct functional layer and warns against overusing glass effects. Formula Vision should use that same hierarchy: rich race content underneath, lightweight chrome above it.

### 3. Game flow, not web conversion

A racing interface should optimize for entering a race, understanding state, switching context, and staying in flow. It should not resemble a marketing landing page with a hero, feature cards, and repeated calls to action.

### 4. Motion must communicate

Every motion belongs to one of three classes:

- **Navigation motion**: where did the view come from and where did it go?
- **Data motion**: what changed in the race state?
- **Ambient motion**: what makes the world feel alive while idle?

No bouncing everything. No unrelated hover gimmicks. Motion is restrained until it has a job.

### 5. Strong identity without sacrificing familiarity

Buttons behave like buttons. Search behaves like search. Playback behaves like playback. The distinctiveness comes from composition, typography, geometry, motion, and race-specific data rather than inventing unfamiliar interaction conventions.

### 6. Density is allowed

The viewer is an analysis product. Dense information is desirable when the hierarchy is strong. We should not simplify timing/telemetry into giant empty cards just to look minimal.

### 7. Page hierarchy beats local polish

A component can be excellent in isolation and still weaken the product when every
neighbor is also trying to be memorable. Review the whole screen after every
major visual change.

Before merging a new visual treatment, ask:

- What is the single dominant object or idea on this screen?
- Is this component helping that hierarchy or competing with it?
- Is the treatment communicating state, identity, navigation, or real data?
- Does the same visual job already have an established treatment elsewhere?
- Would removing the effect make the product less understandable or only less
  decorated?

### 8. Anti-template guardrails

Formula Vision must not converge on a generic generated-product vocabulary.

- Do not use slogan fragments or punctuation cadence only to sound premium.
  Headlines should describe the product directly.
- Do not pair an eyebrow, headline, subtitle, stats strip, badge, and CTA by
  default. Each layer must add new information.
- Pills are reserved for compact bounded controls or modes. Simple archive
  filters should use a flatter index/tab treatment.
- Glows must originate from meaningful race identity, telemetry, or focus
  state. Never add a free-floating decorative glow just because a surface feels
  empty.
- Gradient lighting must describe depth, state, or race identity. It is not the
  default background treatment.
- Avoid card-inside-card composition when whitespace, alignment, or a hairline
  can communicate the same grouping.
- Avoid generic online/status chips when the state does not materially change
  the user's decision.
- One interaction gets one motion story. A hover should not independently
  bounce, rotate, shimmer, glow, and slide multiple unrelated children.
- Frequent focus feedback should be brief and precise. Longer cinematic motion
  is reserved for entering a race or presenting race content.
- Collection widths are deliberate. Editorial feature surfaces can stay
  narrower while the race library uses more of the viewport to improve
  browsing density and prevent clipped cards.
- At 1920px desktop width, the race library should fit six complete 260px race
  cards beside the season rail before horizontal scrolling is necessary.

### 9. Authored-product review gate

Every substantial UI pass should be reviewed at minimum at 1440px and 1920px
desktop widths, plus one narrow viewport.

The review must include:

1. **Squint test:** the primary content should remain obvious when fine detail is
   ignored.
2. **Silhouette test:** large surface shapes should feel intentional without
   relying on copy.
3. **Chrome test:** controls should recede when idle and become obvious when
   relevant.
4. **Repetition test:** repeated cards or controls should share one interaction
   grammar.
5. **Removal test:** delete decorative elements one at a time. If hierarchy
   improves, the element did not earn its place.
6. **Edge test:** focus expansion, scrolling, and sticky chrome must be checked
   at the first and last visible items, not only at center items.
7. **Truth test:** visual identity may change presentation, never canonical race
   geometry or data.

---

## Visual identity

### Typography

Replace Inter.

Recommended stack:

- **Display / race titles:** Barlow Condensed Variable. Tall, mechanical, motorsport-friendly, highly legible, not faux-futuristic.
- **Interface / body:** Instrument Sans. Neutral enough for dense UI but more editorial and distinctive than default AI stacks.
- **Timing / telemetry / numerics:** IBM Plex Mono with tabular numerals.

Rules:

- Race names can be large and condensed.
- Body copy uses sentence case.
- Uppercase is reserved for race codes, status flags, tyre compounds, compact HUD labels, and very small metadata.
- Remove wide letter spacing from normal actions.
- Numerics use tabular figures.
- Do not use more than three weights per family.
- Headline line-height is tight; body line-height is generous.
- Avoid large blocks of center-aligned body copy.

### Color

Base surfaces remain dark, but with fewer competing shades.

Proposed semantic palette:

- Canvas: #070809
- Canvas raised: #0B0D0F
- Panel: #101317
- Hairline: rgba(255,255,255,.08)
- Strong hairline: rgba(255,255,255,.16)
- Primary text: #F5F6F7
- Secondary text: #A2A7AD
- Muted text: #666C73
- Formula Vision red: #FF3B30

Red is not a general decoration. Use it for:
- primary selection/action;
- Formula Vision brand moment;
- critical race state when semantically appropriate.

Race-specific accent colors belong primarily to track content and driver/team data. Status colors remain semantic: yellow, green, red, purple, blue, etc.

### Geometry

Stop giving every surface the same rounded rectangle.

Use three radius tokens only:

- 4px: telemetry/data cells;
- 10px: buttons, compact popovers, controls;
- 18px: major media surfaces/cards.

Large cinematic surfaces can be clipped with subtle asymmetric corners or a custom chamfer, but should not become sci-fi decoration.

Hairlines and spacing should do more hierarchy work than shadows.

### Iconography

Create a tiny Formula Vision icon set instead of leaning on generic library glyphs everywhere:

- play/pause/seek;
- full screen;
- search;
- race switch;
- telemetry;
- timing;
- strategy;
- track;
- result/podium;
- follow camera.

The icon language should use consistent stroke weight and one subtle motif: a clipped apex / telemetry node.

### Brand mark

The current speed-stripe mark is serviceable but too close to generic racing language. Later in the pass, explore a compact FV monogram built from a continuous circuit/apex line. It must work at 16px and in one color.

---

## Motion system

Use Framer Motion already in the project; do not add a heavy animation framework.

### Timing families

- Micro feedback: 90–140ms
- Micro hover/focus feedback: 160–220ms
- Race-tile focus morph: ~320–360ms perceived spring duration
- Panel/layout transition: 240–360ms
- Shared-element race transition: 380–520ms

Primary easing:
`cubic-bezier(.2,.8,.2,1)`

Springs should be used where the interface is expressing spatial continuity,
including timing-row re-ordering, shared-element transitions, and the race
tile's explicit focus morph. Ordinary buttons should not bounce.

### Hover behavior

Hover should reveal information and depth through one coherent focus morph,
not a pile of unrelated effects.

Race-tile focus morph:
- the carousel slot remains fixed so neighboring races never reflow;
- the visual surface grows about 8–9% and rises only as part of that expansion;
- the rail reserves explicit focus overscan and edge cards bias inward only when
  needed, so the enlarged state never looks clipped by its scroll viewport;
- the card background, edge contrast, and circuit scale resolve together;
- the library-card circuit color trace starts after focus intent and completes
  in about 1.3 seconds, slightly slower than the surface morph;
- the focus edge is created with overlay opacity / inset framing rather than
  animating border width;
- the circuit settles into a stronger perspective pose rather than entering an
  infinite hover orbit;
- one specular sweep may run on focus entry, but it must not loop;
- carousel navigation arrows remain visually unchanged but are absent from the
  resting composition. They appear only while the race rail itself is hovered
  or an arrow has explicit keyboard focus.

Pointer parallax:
- maximum ~2deg rotation;
- subtle internal depth separation;
- return is spring-damped and slower than raw pointer movement.

Keyboard focus must trigger the same information state without pointer parallax.

### Reduced motion

`prefers-reduced-motion` gets:
- static track previews;
- no orbit;
- no card tilt;
- crossfades instead of shared-element travel;
- timing order still updates without spring travel if necessary.

---

## Landing / race library

The current SaaS-like hero + three-column card grid should be replaced.

### First fold

Use a media/game-home composition rather than a marketing page.

Left:
- small Formula Vision wordmark;
- one concise headline, e.g. **"Every lap. Reconstructed."**
- one supporting sentence maximum;
- library status / seasons / race count as quiet metadata.

Right:
- a real circuit object from the most recent available race;
- circuit sits on a shallow 3D plane;
- the featured surface can use the same restrained focus morph as race tiles;
- line glow is extremely restrained;
- tiny telemetry particles or one moving locator dot can imply replay without pretending the race is currently playing.

The hero background should mostly be the canvas, circuit geometry, and a subtle light field. Remove the generic grid + radial-blob treatment.

### Library structure

Default desktop layout becomes **season shelves**, not one uniform card grid.

- Each season is its own horizontal rail.
- Sticky/anchored year at the left edge.
- Race tiles scroll horizontally with snap.
- Current/newest season is open first.
- Search filters across every season.
- A compact season navigator can jump between years.
- Mobile collapses into a vertical list grouped by season.

This scales far better when the archive reaches 2018-current and feels closer to a console/media library than a SaaS catalog.

### Race tile

Each tile is a small interactive race poster, not a generic information card.

Resting state:
- race name;
- location;
- year + round;
- large ghosted round number or circuit code;
- simplified circuit silhouette;
- thin accent edge derived from circuit/country theme;
- optional summary from catalog: laps, drivers, duration.

Hover/focus state:
- the visual card surface expands smoothly within a fixed carousel slot;
- circuit becomes the dominant layer and grows with the surface;
- path gains depth, resolves into a stronger perspective pose, and is traced in
  accent color after the surface gains focus;
- edge contrast thickens perceptually through layered 1px frames, not changing
  box-model border width;
- background tone and accent light deepen while the information architecture
  remains the same;
- a restrained "Enter race" action becomes higher contrast;
- no giant glow, no looping orbit, and no elastic/cartoon bounce.

### Track preview implementation

Do not use WebGL for the first version.

1. Use IntersectionObserver to preload published geometry for cards near the
   viewport, while still avoiding an all-at-once archive fetch.
2. Resolve the race manifest through the existing immutable library helper.
3. Read `manifest.metadata.track_points`.
4. Normalize to a small SVG viewBox.
5. Simplify the path to roughly 80–160 useful points if needed.
6. Render a single SVG path.
7. Put the SVG in a CSS 3D scene:
   - `perspective(...)`
   - `rotateX(...)`
   - a deterministic focused transform on hover/focus rather than a looping
     orbit;
   - one moving specular/light sweep.
8. Cache the manifest/path per race.
9. Do not preload every manifest at page load.

Resting visible cards should converge on the real published
`manifest.metadata.track_points`, not remain on a procedural placeholder until
hover. The hover interaction controls lighting and focus, not whether circuit
geometry is truthful.

Later, if manifest latency becomes noticeable, add a small `track_preview`
payload to catalog generation so the landing page can render circuit
silhouettes without extra requests.

### Search/filter

Search becomes part of the library chrome, not a rounded "form card."

Desktop:
- persistent search icon + input;
- season chips only when needed;
- `/` shortcut can focus search later.

The UI should keep the archive visible while filtering. No large empty card when there are zero results; use a clean in-context empty state.

---

## Race entry transition / briefing

The current centered `PreRaceCard` feels like a web modal. Replace it with a **Race Briefing** transition.

When a race is chosen:

1. The selected tile expands through a shared `layoutId`.
2. Its circuit silhouette becomes the central track object.
3. The page background falls away.
4. Race name, location, lap count, duration, and driver count settle into a low data rail.
5. The primary control becomes **Start replay**.
6. Once the canonical graph is ready, the track object transitions directly into the viewer's track map.

This creates object permanence: the user chose a race object and sees that same object become the replay.

Loading during briefing should use real structure:
- track outline draw if metadata is already available;
- small acquisition/status indicator otherwise;
- never fake a percentage.

The existing fail-closed semantics remain intact. Integrity detail can move into a "Details" disclosure instead of a long paragraph in the primary UI.

### Race identity contract

Race identity is shared between library cards and the briefing:

- each known race/circuit has one stable curated accent;
- unknown races receive a deterministic accent from a restricted non-white
  palette, so identity remains stable across reloads and seasons;
- the briefing track uses accent-only strokes and glow, while its pulsing
  locator remains white;
- the briefing reveal is intentionally a little faster than library-card
  drawing, about 1.12 seconds, with a dim structural line, an animated
  accent-glow pass, and a crisp final trace. The extra light exists only during
  the reveal so the steady state stays restrained;
- the briefing ambient field and small metadata accents inherit the same race
  accent;
- controls, headlines, and body copy remain predominantly monochrome;
- country identity appears as one compact local SVG flag beside the circuit
  location, never as a separate decorative panel. Local assets are used instead
  of Unicode flag emoji so Windows/Chromium rendering is deterministic;
- theme changes never alter canonical track geometry or replay data.

---

## Viewer redesign

The viewer should feel like a professional race workstation crossed with a premium game HUD.

### Overall hierarchy

1. **Track / race state** is primary.
2. **Timing tower** is always-readable authority.
3. **Playback/timeline** is persistent transport.
4. **Driver/strategy detail** appears contextually.
5. Navigation recedes.

### Top chrome

Reduce the current app header height and visual weight.

Landing:
- mark/name left;
- GitHub/About right.

Viewer:
- back/library affordance left;
- race identity center;
- race switcher right;
- translucent/raised material allowed here because this is control chrome.

Avoid a permanent heavy black bar if the track can extend behind the chrome while preserving readability.

### Race-control ribbon

Race control becomes a contextual ribbon at the top of the race canvas.

Normal state:
- very quiet.

Yellow/VSC/SC/red:
- expands slightly;
- status color becomes stronger;
- relevant message is shown;
- animation reflects importance without flashing continuously.

### Timing rail

Rebuild the tower visually while keeping the same timing authority.

Each row:
- position;
- thin team/driver color marker;
- driver code/name;
- tyre compound and age;
- interval;
- optional lap-time context.

Changes:
- reduce nested rounded badges;
- make tyre indicator a compact round/square glyph;
- position changes animate spatially;
- selected/followed driver gains a clear structural state, not only a background tint;
- fastest lap uses purple as a semantic accent, not glow decoration.

Density settings can come later, but the default should fit the field without feeling cramped.

### Track canvas

The track should own more of the screen.

Default:
- clean top-down or very mild perspective;
- drivers on track;
- active driver gets a focus halo/label;
- selected driver's previous 1–2 seconds can get a subtle motion trace;
- other cars recede.

Later "Track Lens" can add:
- corner labels;
- speed zones;
- DRS zones;
- sector overlays;
- incident markers.

Do not distort race positions in ways that imply timing authority comes from the GPS rendering.

### Driver inspector

The right side should not permanently consume space when no driver is selected.

Closed:
- track expands.

Open:
- contextual inspector slides in;
- driver name, number, team;
- current P/gap/tyre;
- speed, gear, throttle, brake, DRS;
- lap/sector comparisons;
- strategy/stint visualization.

Use charts as instruments, not white-box dashboard cards.

### Transport dock

The bottom becomes one integrated transport object.

Core:
- play/pause;
- skip;
- current lap + elapsed time;
- speed;
- results;
- full timeline.

Timeline:
- lap boundaries;
- pit events;
- fastest laps;
- safety car/VSC/red flag;
- race-control markers;
- scrub preview.

A user should be able to understand the race's story from the timeline alone.

### Lenses

Do not ship all lenses at once, but reserve the architecture now:

- TRACK
- DUEL
- STRATEGY

Track is current spatial replay.
Duel will compare two drivers.
Strategy will focus on stints, tyre life, pit windows, and timing deltas.

These should be modes of the same race, not separate pages.

---

## Results

Results should feel like the race resolving, not a generic modal table.

- Podium or top-three treatment at top.
- Full classification below.
- Position gain/loss.
- fastest lap;
- tyre/stint summary where useful;
- driver row can reopen the inspector.
- "Replay from start" and "Back to library" are clear final actions.

Use the same typography and timing hierarchy as the live race.

---

## Copy and voice

Formula Vision copy should be short, confident, and literal.

Rules:
- no generic tech slogans;
- no unnecessary em dashes;
- no "unlock", "supercharge", "seamless", "next-generation", or similar SaaS filler;
- active verbs for actions;
- sentence case for most interface text;
- avoid explanatory paragraphs when layout can explain the state.

Examples:

Instead of:
"Historical race intelligence"

Use:
"Race archive"

Instead of:
"Full-race spatial replay, authoritative timing, strategy context, race control and driver telemetry: synchronized into one interactive view."

Use:
"Timing, telemetry and every lap in one replay."

Instead of:
"Open replay"

Use:
"Enter race" or simply a play/arrow affordance once the card state makes the action obvious.

---

## Loading, empty, and error states

High-quality products are recognizable in non-happy states.

### Loading
- custom track/data skeletons;
- avoid generic six rounded shimmer cards;
- show content geometry early;
- keep UI interactive where possible.

### Empty search
- in-context message;
- one reset action;
- do not create a giant dashed rectangle.

### Catalog failure
- concise title;
- retry;
- details disclosure with immutable/fail-closed explanation;
- do not overwhelm a nontechnical visitor with pipeline language.

### Race artifact failure
- preserve strict failure;
- identify which race failed;
- offer return to library and retry;
- technical detail can be copyable under disclosure.

---

## Accessibility

Visual ambition cannot depend on hover, color, or motion.

- Hover states duplicated by keyboard focus.
- Touch gets a strong static track preview instead of hover choreography.
- Every status color has text/icon shape support.
- Maintain `prefers-reduced-motion`.
- Maintain strong focus visibility, but redesign it to match the system.
- Timing numerics preserve contrast at small sizes.
- No critical information only in tooltips.
- Build room for a "reduced effects" setting later.
- Consider HUD density/opacity controls later, borrowing the useful idea from racing games that let players tune OSD elements.

---

## Responsive behavior

### Desktop wide
Full timing rail + track + optional inspector + transport dock.

### Laptop
Narrower timing rail; inspector overlays rather than permanently shrinking track.

### Tablet
Timing becomes collapsible drawer; track remains primary.

### Mobile
The goal is not to reproduce a desktop broadcast layout. Use:
- track;
- compact top race state;
- swipe/segmented switch between timing, track, driver;
- bottom playback;
- no hover-only interactions.

---

## Technical architecture

Keep React, Vite, Tailwind, Framer Motion, and Recharts. Do not introduce a component library whose default styling would become the new visual identity.

Add a product-specific UI layer:

- `designTokens.ts` or Tailwind theme tokens;
- CSS variables for semantic color, radii, elevation, and motion;
- `TrackPreview`;
- `RaceTile`;
- `SeasonRail`;
- `RaceBriefing`;
- `ViewerChrome`;
- `RaceControlRibbon`;
- `TimingRail`;
- `DriverInspector`;
- `TransportDock`;
- `StatusGlyph`;
- `DataValue`.

Prefer components that encode Formula Vision semantics over generic `Card`, `Badge`, and `Panel` abstractions.

### Catalog/session enrichment

The published catalog already contains:
- duration;
- laps;
- driver count;
- capabilities;
- source quality.

The frontend currently discards most of these when mapping catalog entries into `Session`. Extend the landing model so race tiles can display useful real data without another request.

Track geometry can initially be lazy-loaded from the immutable race manifest and later promoted into a lightweight catalog preview field if needed.

---

## Performance budget

A premium UI that drops frames is not premium.

Landing:
- no WebGL required for circuit card effects;
- no loading all manifests at once;
- hover fetch after a short intent delay;
- cache every resolved preview;
- use IntersectionObserver/requestIdleCallback only for a small number of likely-next races;
- use `content-visibility` for offscreen season rails where practical.

Motion:
- transform/opacity first;
- avoid animating layout-heavy properties;
- SVG path effects should stay lightweight;
- pointer parallax updates through requestAnimationFrame.

Fonts:
- self-host only the weights actually used;
- variable font where it reduces payload;
- preload the primary interface font.

Target:
- 60fps interactions on a normal laptop;
- no long task over ~100ms from hover animation;
- no race-card interaction that blocks navigation on a network request.

---

## Anti-vibe-coded quality gate

A PR should fail design review if it introduces a cluster of the following without a strong product-specific reason:

- Inter/DM Sans/default geometric-sans everywhere;
- purple/blue gradient as a prestige shortcut;
- glowing glass cards;
- identical radius on every object;
- generic three-column feature grid;
- centered hero + pill + CTA + dashboard screenshot formula;
- arbitrary Tailwind values with no token;
- one-off component styling that contradicts another screen;
- uppercase tracked copy for ordinary labels;
- "modern/premium" decoration with no interaction or information job;
- hover = translateY + box shadow;
- multiple competing animation styles;
- placeholder loading/error states;
- decorative motion that degrades race readability.

The substitution test: if Formula Vision's name and red accent were replaced by another startup's name/color and the screen would still make sense unchanged, the screen is not specific enough.

---

## Implementation sequence

### Phase 0: system foundation
- install/bundle the chosen font families;
- build semantic tokens;
- replace global Inter;
- establish radii, type scale, spacing, motion timings;
- create base data/interaction primitives;
- define visual regression screenshots for landing + viewer.

### Phase 1: race library
- build SeasonRail + RaceTile;
- expose catalog laps/driver count/duration;
- build TrackPreview from canonical manifest geometry;
- implement hover/focus parallax + orbit;
- replace hero and archive grid;
- rebuild search/filter chrome;
- custom loading/empty/error states.

This phase should create the biggest "holy shit" improvement fastest.

### Phase 2: race briefing
- shared element transition from RaceTile;
- cinematic circuit object;
- readiness state;
- direct handoff into viewer.

### Phase 3: viewer shell
- new chrome;
- race-control ribbon;
- TimingRail;
- larger track canvas;
- integrated TransportDock/timeline.

### Phase 4: contextual analysis
- DriverInspector;
- improved telemetry visualization;
- results;
- strategy visuals.

### Phase 5: product-level polish
- responsive passes;
- reduced-effects mode;
- controller/gamepad navigation exploration;
- optional restrained UI audio;
- custom brand mark/icon set;
- final motion tuning;
- performance/contrast/keyboard audits.

---

## First implementation target

Start with **Phase 0 + Phase 1**, not the viewer.

Why:

1. The race library is the first impression and currently contains the strongest generic-web tells.
2. It gives us a controlled place to establish typography, motion, geometry, and the race-specific visual language.
3. Track-preview hover is a memorable product-specific interaction that directly uses Formula Vision's underlying race data.
4. Once the system feels right on the library, the viewer can inherit it instead of being redesigned ad hoc.

The first "design complete" checkpoint should be one landing screen that makes a stranger want to click a race before they understand the engineering.

## Reference research

These are principle references, not templates to clone:

- Apple HIG, Branding and Liquid Glass: content-first hierarchy, restrained accent color, custom type where justified, controls/navigation as a separate functional layer.
  - https://developer.apple.com/design/human-interface-guidelines/branding
  - https://developer.apple.com/documentation/TechnologyOverviews/liquid-glass
  - https://developer.apple.com/documentation/TechnologyOverviews/adopting-liquid-glass
- Apple HIG, Layout / Feedback / Loading: hierarchy, immediate feedback, custom game loading states.
  - https://developer.apple.com/design/human-interface-guidelines/layout
  - https://developer.apple.com/design/human-interface-guidelines/feedback
  - https://developer.apple.com/design/human-interface-guidelines/loading
- Spotify Encore / Spotify design history: connected tokens for type, motion, spacing, writing/accessibility; media content carries emotion while product chrome stays consistent.
  - https://engineering.atspotify.com/2021/4/customization-vs-configuration-in-evolving-design-systems
  - https://newsroom.spotify.com/2026-04-23/spotify-design-history/
- Linear 2026 refresh: do not make every element compete for attention; preserve rich information density while allowing secondary chrome to recede.
  - https://linear.app/now/behind-the-latest-design-refresh
- Returnal UI/UX: gameplay-first hierarchy; intentional animation timing/color; interface art direction tied to the game's world.
  - https://blog.playstation.com/2021/05/11/unpacking-returnals-ux-design-gameplay-first-ui-retro-futuristic-tech-and-accessibility/
- Need for Speed / Forza UI work by Territory Studio: menu/HUD motion, image-led emotional front ends, and fresh visual language balanced against familiar racing information expectations.
  - https://territorystudio.com/project/need-for-speed/
  - https://territorystudio.com/project/forza/
- Metaphor: ReFantazio UI: each interface can carry its own memorable composition while preserving unity; interface design can be part of emotional/game identity, not just invisible chrome.
  - https://gdcvault.com/play/1035332/From-Persona-to-Metaphor-ReFantazio
- EA F1 accessibility resources: useful precedent for HUD visibility/customization and colorblind/high-visibility options.
  - https://www.ea.com/able/resources/f1-25
- Research on vibe-code design homogenization:
  - https://arxiv.org/abs/2603.13036
  - https://blog.interfacekit.io/what-makes-a-website-look-ai-generated
  - https://blog.interfacekit.io/why-vibe-coded-websites-look-the-same

