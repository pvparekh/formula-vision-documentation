# Formula Vision Spatial Camera Architecture

## Goal

Evolve Formula Vision from a top-down replay into a circuit-relative 3D race viewer.

The core idea is simple: **the track is the coordinate system**.

Every camera, driver pose, race overlay, braking zone, duel view, and future broadcast camera should be derived from one normalized spatial track model instead of hard-coded per-circuit behavior.

This makes one implementation work across Monaco, Suzuka, Spa, Las Vegas, and every other validated circuit.

## Existing data advantage

Formula Vision already has most of the difficult source data.

Canonical position telemetry stores:

- X
- Y
- Z
- session time
- source quality
- driver identity

The browser contract currently publishes X/Y only, and the present SVG replay treats the circuit as a two-dimensional coordinate plane.

That means the expensive acquisition step does **not** need to be repeated to experiment with elevation. Existing canonical snapshots can later be rebuilt into newer immutable browser releases that expose Z and richer spatial metadata.

FastF1 describes position X/Y/Z in 1/10 meter units. Formula Vision should normalize these to meters inside the spatial rendering layer.

Z must be quality-gated. Some sessions may contain flat or low-quality altitude. The 3D mode must remain excellent with X/Y only by rendering a flat track in perspective; true elevation is an enhancement, never a requirement.

## Spatial model

### Closed centerline

Build one closed circuit centerline from the validated track polyline.

Represent progress as arc length:

- `s = 0` at the start of the reference polyline
- `s` increases around the circuit
- values wrap at total circuit length

This gives the application a circuit-relative coordinate independent of screen pixels.

### Local track frame

At every sampled point derive:

- point `P(s)`
- tangent `T(s)`
- lateral normal `N(s)`
- world up `U`
- curvature
- optional elevation

This is a simplified Frenet-like frame.

Three.js itself exposes curve arc-length sampling and Frenet-frame generation; the final renderer can use the same concepts directly for smooth track geometry.

### Driver projection

Each interpolated driver position is projected to the closest segment on the track.

Naive nearest-neighbor projection is insufficient because tracks can approach or cross themselves. Suzuka is the obvious example.

Projection therefore uses temporal continuity:

1. project around the driver's previous `s`
2. prefer a bounded arc-length search window
3. fall back to global nearest projection after seeks, source gaps, or rejoin events

The projection yields:

- circuit progress
- local tangent
- local normal
- curvature
- lateral distance from reference centerline

The implementation foundation lives in `frontend/src/lib/spatialTrack.ts`.


## Current implementation status

The additive spatial follow system is now implemented on the milestone-6 branch.

### Shipping now

- **MAP remains the default.** The original top-down renderer and its existing driver-follow behavior are preserved.
- **CHASE is opt-in.** A compact `MAP | CHASE` control switches renderer modes without changing replay time.
- Switching from MAP follow to CHASE preserves the same followed driver.
- Switching back to MAP preserves that driver, so the user returns directly to the proven top-down follow view.
- Entering CHASE without an existing selection chooses the best available target: current follow, selected driver, current leader, then the first published driver.
- Driver selection from the timing tower becomes camera-aware while CHASE is active.
- CHASE is lazy-loaded as its own bundle, so MAP does not pay the spatial renderer cost.
- The current CHASE renderer uses a custom perspective projection and world-width SVG track ribbon. This keeps the first implementation small, deterministic, and isolated from the proven SVG map.
- Track width is represented in world units before projection, so the circuit widens toward the camera and narrows with distance instead of looking like a uniformly thick tilted polyline.
- Driver markers are projected into the same camera space and orient from the local circuit tangent.
- Camera motion is based on circuit progress rather than raw screen coordinates.
- Camera distance, height, lateral offset, and look-ahead react to spatial speed and local curvature.
- Position-derived speed is used only as a presentation input for camera dynamics; it is **not** displayed as authoritative telemetry.
- Camera position and target use independent frame-rate-independent smoothing.
- Seeks, follow-target changes, and large replay jumps reset camera smoothing so stale momentum cannot drag the camera across the circuit.
- Projection uses previous circuit progress as a continuity constraint to handle close/overlapping geometry such as Suzuka.
- Unusable track geometry or a sustained inability to resolve the followed car automatically returns the user to MAP.
- Existing race control, timing, telemetry inspector, playback, results, and timeline remain shared across both camera modes.
- The map-only quick guide is hidden in CHASE.
- Escape still returns to the stable MAP state and releases camera focus.
- Reduced-motion behavior remains controlled by the application-wide accessibility treatment.

### Elevation

The canonical acquisition layer already preserves source X/Y/Z position telemetry.

The web-artifact builder now also preserves optional Z in the derived circuit polyline. The browser spatial model:

- requires broad Z coverage;
- rejects effectively-flat, implausibly large, or spiky altitude;
- smooths accepted elevation around the closed circuit;
- otherwise falls back to a clean flat spatial stage.

Current already-published releases that contain X/Y-only track points still work in CHASE. Their circuit renders on a flat 3D plane with perspective.

Canonical snapshots do **not** need to be re-downloaded to upgrade an older race to an elevation-aware web release; the richer browser artifact can be rebuilt from the stored canonical snapshot later.

### Bundle impact

CHASE remains lazy-loaded as an independent frontend chunk so MAP and the default
race experience do not pay the spatial-renderer cost until CHASE is opened.

Bundle size is treated as a measured build artifact rather than a hard-coded
architecture constant. Record the current raw/gzip values from the production
build whenever a spatial pass materially changes the renderer.


## Rendering approach

Use Three.js through React Three Fiber for the 3D stage.

The current React/Vite architecture can keep all timing, telemetry, selection, playback, and race-control state. Only the spatial canvas changes renderer.

Recommended packages when the visible prototype begins:

- `three`
- `@types/three`
- `@react-three/fiber`
- optionally `@react-three/drei` for mature primitives/controls

Do not put the entire application into WebGL. Timing, transport, results, menus, and inspector remain DOM UI.

### Why this renderer

Three.js provides:

- PerspectiveCamera
- CatmullRomCurve3
- arc-length curve sampling
- Frenet frames
- TubeGeometry / custom BufferGeometry
- instancing
- GPU rendering

React Three Fiber integrates the render loop and scene with existing React state while supporting adaptive DPR and performance controls.

## Track art direction

Do not attempt photorealistic Google Earth.

Formula Vision should render a stylized **3D telemetry stage**:

- dark world
- physical-looking track ribbon
- subtle track edge
- minimal terrain/ground plane
- team-color driver markers or abstract open-wheel silhouettes
- event lighting only when semantically meaningful

This avoids pretending we have authoritative barriers, buildings, kerbs, banking, or terrain meshes when our source does not provide those.

### Track mesh

Initial implementation:

1. smooth canonical track points using a closed centripetal Catmull-Rom spline
2. sample by arc length
3. build a flat ribbon mesh around the centerline
4. use local normal vectors for left/right track edges
5. add true Z only when elevation quality passes
6. optionally add a thin under-surface so the track reads as a physical object

A constant visual width is preferable to inventing exact real-world width.

## Camera modes

### MAP

Existing top-down replay.

This remains the clearest whole-field analytical view.

### FOLLOW

Existing top-down focused camera.

Keep as a lightweight analytical follow mode.

### CHASE 3D

New project-defining mode.

Camera:

- behind selected driver
- elevated above track
- looks 25–60 m ahead depending on speed
- orientation comes from local track tangent, not noisy raw GPS heading
- driver remains in lower-middle composition rather than exact center
- camera damping is frame-rate independent

This should feel like a tactical chase camera, not a racing-game cockpit camera.

### HELI

High oblique follow inspired by aerial broadcast/helicopter tracking.

- higher altitude
- lateral offset
- wider field of view
- more of the next corner visible
- excellent default for users who want "Google Maps 3D" perspective without being too close

### DUEL

Frame two selected drivers.

Camera target is midpoint between them.
Distance/FOV expands and contracts so both remain visible.
Their timing delta and telemetry can live around the canvas.

This uses the same spatial engine; it is not a separate map implementation.

### BROADCAST

Future mode.

Generate virtual camera nodes around the circuit from geometry:

- corner entry
- apex/high-curvature regions
- corner exit
- long straights
- braking zones

The active camera changes based on selected driver's progress.

No circuit-by-circuit authoring is required for the first version, though hand-tuned overrides can later improve major circuits.

## Single-driver CHASE presentation

CHASE intentionally renders only the selected driver.

The full field remains available in MAP, where spatial comparison is useful.
Rendering every raw driver sample inside CHASE created visual clutter and
amplified positional noise through perspective projection.

The selected driver now has four deliberately separate spatial concepts:

1. **Authoritative lap progress** still comes from source position telemetry
   projected onto the validated circuit and stabilized by the longitudinal
   phase model. No presentation physics can change timing, lap position, replay
   ownership, or seek behavior.
2. **Geometric racing-line intent** is a circuit-generic target generated from
   resolved corners, smooth track geometry, track width, linked-corner logic,
   and a reference-speed feasibility pass.
3. **Realized CHASE motion** is a separate time-domain state. The car does not
   teleport onto the geometric target. Actual replay speed, longitudinal
   acceleration, local road curvature, a broad F1 grip envelope, lateral
   velocity limits, lateral acceleration limits, and lateral jerk limits decide
   how quickly the car is allowed to move toward that target.
4. **Off-line / pit / incident fallback** remains source-relative. When the
   source position is more than 7.25 m from the validated centerline, CHASE
   stops normalizing onto the racing line and returns to the presentation
   centerline plus the full filtered source lateral offset.

This distinction matters because spatial smoothness alone is insufficient.
A path can be perfectly C2-smooth in distance and still look violently wrong at
high speed: the same 8 m cross-track move that takes 2.5 seconds at low speed
may take well under a second at 300 km/h. The previous implementation bounded
`d(offset)/ds` but not the corresponding time-domain motion strongly enough.
The current architecture constrains **both** the path in space and the realized
motion in time.

The geometric line builder remains circuit-generic:

- official FastF1 corner metadata is used when available; the existing
  curvature-derived fallback supplies corner hints for older releases;
- each hinted corner is refined within a bounded local window to the strongest
  matching smoothed-curvature peak;
- apex offset moves toward the inside edge while preserving roughly a
  car-width-plus-margin inside the rendered road;
- tighter corners receive a small bounded late-apex bias;
- when enough straight exists between corners, the path explicitly tracks out
  after the apex and sets up the next corner from its outside edge;
- close opposite-direction sequences are treated as one compound trajectory,
  not as two isolated outside-apex-outside corners glued together;
- linked apex amplitudes are reduced when the available distance cannot support
  the requested cross-track sweep;
- nearby same-direction corners can share a partial track-out;
- anchor transitions use quintic smooth interpolation;
- tangent, normal, and curvature are recomputed from the generated line itself.

The new **reference-speed feasibility pass** then regularizes that desired line
before it ever reaches the renderer. It estimates a conservative local speed
from centerline curvature using a 4.6 g reference lateral acceleration, caps the
estimate to 30-95 m/s, converts a 5.8 m/s cross-track velocity budget into a
local `d(offset)/ds` limit, and applies repeated forward/backward projection
around the closed circuit. High-speed sections therefore spread a lateral
transition over more distance, while slow hairpins retain stronger
outside-apex-outside use of the available road. This pass is deterministic and
driver-independent; it is a geometry safety rail, not a vehicle model.

The realized CHASE layer then uses the selected driver's actual replay dynamics:

- speed and acceleration come from canonical telemetry when available;
- because this is historical replay rather than a live feed, speed, RPM, and
  throttle are linearly reconstructed between adjacent telemetry samples when
  the source gap is <= 1 second instead of exposing a sample-and-hold staircase;
- gear, brake, and DRS remain stepwise authoritative states;
- a speed-dependent look-ahead grows from roughly 0.18 s to 0.72 s (bounded to
  6-72 m), so a fast car begins preparing for a lateral transition earlier
  instead of discovering the apex too late and darting across the road;
- the desired racing-line amplitude is reduced if the target curvature at the
  current speed would exceed a broad 5.8 g combined-acceleration presentation
  envelope after accounting for longitudinal acceleration;
- lateral target tracking is explicitly bounded by speed-dependent lateral
  velocity, acceleration, and jerk;
- the motion integrator substeps in replay time at up to 60 Hz internally, so
  0.5x, 1x, 2x, 4x, and irregular browser frame rates do not change the
  qualitative vehicle response;
- local road curvature and longitudinal acceleration consume part of the same
  broad grip budget before synthetic cross-track acceleration is allowed;
- chassis heading is reconstructed from the realized lateral velocity divided
  by longitudinal speed, so the body points along the motion it is actually
  making instead of instantly adopting the target-line tangent;
- realized curvature adds the bounded lateral-acceleration contribution
  approximately as `a_lat / v^2`, keeping front-wheel steer coupled to the
  motion the viewer sees;
- the chase camera aims along a short extrapolation of the realized lateral
  position, velocity, and acceleration rather than looking directly at the raw
  geometric racing-line target. Camera motion therefore cannot re-amplify a
  lateral correction that the chassis physics layer already rejected.

The 5.8 g value is intentionally a **presentation guardrail**, not a claim that
Formula Vision has recreated an F1 tyre model. Formula 1 documents sustained
high-speed corner loads around 4-5 g with peaks around 5.6 g at circuits such as
Mugello. Real grip also depends on aero load, tyre state, temperature, compound,
ride height, wind, kerbs, surface state, setup, and many other variables that
are not present in the browser contract.

This architecture follows a strict missing-data rule: when an authoritative
channel exists, reconstruct it continuously and use it; when it does not exist,
fill the gap with a deterministic bounded model whose constraints are explicit.
Do not fabricate wheel slip, tyre smoke, suspension load, steering torque, or
other dynamics simply because they would look dramatic.

The resulting authority split is:

```text
historical source X/Y + telemetry
          |
          +-------------------------------+
          |                               |
          v                               v
branch-safe progress               continuous channels
(authoritative)                    reconstructed in time
          |                               |
          v                               v
longitudinal phase          speed / accel / brake / throttle
          |                               |
          |                +--------------+
          |                |
          v                v
smooth track geometry -> geometric racing-line intent
                          |
                 reference-speed spatial
                    feasibility pass
                          |
                          v
                speed/grip/jerk-bounded
                 realized lateral state
                          |
          +---------------+---------------+
          |                               |
          v                               v
rigid chassis pose                 realized camera aim
          |
          v
CHASE renderer

clearly off-line source position (> 7.25 m)
          |
          v
presentation centerline + full filtered source lateral offset
(no forced normalization)
```

This is still intentionally a **presentation racing line**, not a minimum-time
solver. The next fidelity ceiling is an offline per-race trajectory product:
align historical driver XY, speed, throttle/brake and lap context into circuit
coordinates, reject pits/incidents/traffic laps, learn or aggregate an
expert-like lateral reference, then run a constrained offline optimizer. The
2026 F1-data trajectory work cited below strongly supports that direction.

### Circuit intelligence and corner annotations

Circuit annotations are shared by MAP and CHASE through
`frontend/src/lib/circuitIntelligence.ts`.

When available, canonical extraction calls FastF1
`Session.get_circuit_info()` and stores optional corner metadata:

- turn number and optional letter;
- X/Y marker coordinate;
- marker angle;
- distance along the reference lap;
- circuit rotation.

That metadata is optional. Acquisition failure produces a warning and never
invalidates an otherwise usable race snapshot.

The browser manifest publishes the optional circuit metadata inline. For older
immutable releases that do not yet contain it, the frontend infers approximate
corner locations from circuit curvature so annotations still degrade
gracefully instead of disappearing.

Common corner names are a curated display layer on top of official turn
numbers. Formula Vision only names corners with widely-used, intentionally
mapped names. All other corners remain `T1`, `T12`, and so on. It never
fabricates a name from geometry.

MAP and CHASE consume the same resolved corner list:

- MAP uses fixed-size HTML labels anchored outside the track, with zoom-based
  priority and label-label collision rejection;
- MAP also suppresses any label whose box would cover a visible driver marker;
- whole-track view shows only named/high-priority corners;
- medium zoom adds major unnamed turns;
- close zoom can reveal the full turn set;
- CHASE shows at most one upcoming/current corner in a screen-side safe zone;
- left-hand corners use the left safe zone and right-hand corners use the right
  safe zone;
- the right-side safe boundary is dynamic: when the driver-detail overlay is
  open, right-hand turn callouts anchor immediately to the left of that panel
  with the same 18 px gutter used against the bare viewport edge;
- MAP and CHASE share a larger top-right camera-focus badge. With driver detail
  open, that badge uses the detail panel as its right edge; with the panel
  closed it returns to the viewport edge while staying aligned with the
  top-left track/spatial title row;
- CHASE callouts are screen-space UI and never float over the road/car;
- the callout appears on approach, can show `APEX` as the car reaches the
  corner, and fades after exit.

### Stability pipeline

The current CHASE stability pass reconstructs a presentation trajectory before
camera lag is applied. The system now treats sparse telemetry reconstruction,
circuit progress, camera motion, and rendering as separate stages rather than
trying to solve all visible jitter with one smoothing constant.

The key implementation choices are:

- canonical MAP coordinates stay on the existing linear interpolation path;
- canonical driver speed, RPM, and throttle are reconstructed continuously
  between adjacent historical telemetry samples when the gap is sane; discrete
  gear/brake/DRS state remains sample-held;
- the presentation racing line is spatially regularized by a
  curvature-estimated reference-speed envelope before runtime;
- the selected car follows that line through a separate replay-time lateral
  state with bounded velocity, acceleration, and jerk instead of being attached
  directly to the geometric spline;
- all lateral physics substep in replay time, not display-frame time;
- high-speed line preparation uses bounded look-ahead, while a combined
  acceleration envelope reduces impossible target curvature;
- chassis tangent and steering curvature are derived from realized motion state,
  not copied directly from the target line;
- camera aim extrapolates the realized lateral state so the camera cannot
  amplify rejected target motion;

- CHASE receives optional presentation-only X/Y coordinates reconstructed with
  cubic Hermite interpolation using neighboring historical samples;
- endpoint velocity estimates are bounded by adjacent secant speeds so noisy
  samples cannot create large spline overshoot;
- Hermite deviation from the source chord is capped as another safety rail;
- raw/presentation XY is still projected through the branch-safe circuit model;
- measured circuit progress is velocity-filtered and corrected gradually;
- repeated replay timestamps do not reset the progress velocity;
- when the replay loader and CHASE requestAnimationFrame loops are temporarily
  out of phase, the latest filtered progress defines a moving target rather
  than directly defining rendered progress;
- rendered circuit progress tracks that moving target with a critically damped
  scalar phase spring, so a newly published replay sample cannot reset
  prediction age and create a visible forward/backward hop;
- camera position and aim use independent per-axis critically damped springs.
  Plan-view motion stays responsive while Z uses heavier damping so elevation
  or source-height noise does not make the viewpoint bob;
- canonical selected-driver speed, brake, and throttle telemetry drive subtle
  camera dynamics when available; canonical speed also drives between-sample
  longitudinal phase prediction after a light speed-state filter, while
  projected track position remains the authoritative phase correction;
- the chase car is no longer a free screen-space billboard. Its vehicle
  footprint is projected from the local road frame, then orthogonalized into a
  rigid 2.5D screen basis. Chassis forward still follows the road tangent and
  lateral orientation follows the road normal, but bounded longitudinal
  foreshortening prevents the SVG silhouette from shearing or stretching as an
  arbitrary affine parallelogram through corners;
- the chassis center has no presentation-only vertical hover offset. During
  normal on-track CHASE the chassis follows the corner-coupled presentation
  racing line with only a 12% source-lateral residue capped at 0.65 m; clearly
  off-line positions instead recover the full filtered source lateral offset;
- chassis forward/pitch is measured across approximately one vehicle wheelbase
  (front/rear path samples) instead of from one infinitesimal tangent, which
  suppresses tiny heading/elevation wiggles using rigid-vehicle geometry;
- front-wheel steer is the no-slip bicycle-model steering angle implied by the
  local road-path curvature, while body sideslip remains zero because Formula
  Vision has no authoritative steering-angle or slip-angle channel;
- the forward guide is generated from the same normalized line as the on-track
  CHASE chassis and begins far enough ahead that it cannot visually pass
  underneath the car;
- source Z is quality-gated more strictly, spatially smoothed, and never
  Catmull-Rom-overshot between elevation controls;
- camera damping is applied to rig offsets relative to the current road/car
  anchor rather than to absolute world position. Track translation/elevation
  therefore carries car and camera together while the rig retains smooth
  relative motion;
- FOV is fixed. Speed-driven camera distance, height, and look-ahead now move
  as one coordinated composition envelope rather than three loosely-related
  values; this keeps the selected car in a much tighter screen-space band on
  straights;
- speed defines that stable chase envelope, while longitudinal acceleration
  drives a separate bounded transient arm displacement: up to about 1.65 m
  farther back under strong throttle/acceleration and about 0.75 m closer under
  braking;
- that transient uses a critically damped scalar spring with deliberately
  slower engagement/reversal/release than the previous pass, so it reads as
  camera mass instead of a zoom pulse or forward/backward tug;
- camera damping advances in replay time, preserving approximately the same
  camera character at 0.5x, 1x, 2x, and 4x, and freezes when replay is paused;
- this follows the same high-level chase-camera principle exposed by mature
  game camera systems: inherit the moving target/vehicle frame, then apply
  bounded lag to the rig rather than independently smoothing the target through
  world space;
- the visible road is a dense local corridor around the selected driver rather
  than a projection of the entire circuit every render frame;
- local road vertices are anchored to fixed global circuit-distance intervals,
  so the road is static world geometry instead of subtly sliding under the car;
- sparse asphalt grain and resurfacing seams are also keyed to global circuit
  distance. They create track-relative optical flow while remaining static
  world geometry; speed only modulates their subtle opacity, never their phase;
- the local road corridor extends well behind the selected car and uses a
  road-specific near-plane projection with large offscreen overscan. Screen
  bounds never terminate ribbon geometry; the SVG viewport performs the final
  visual clip so the asphalt disappears naturally beneath/behind the camera
  instead of exposing the end of the render window;
- each centripetal Catmull-Rom source segment is reparameterized by measured
  curve arc length while preserving its source progress endpoints, preventing
  visual speed pulsing inside spline segments;
- circuit sampling uses binary search because the local corridor and trace
  perform many arc-length samples per frame;
- animated SVG Gaussian-blur filters are avoided in the moving stage in favor
  of layered vector strokes;
- the selected driver is represented by a lightweight generic modern
  open-wheel silhouette colored from the published driver/team color.

The current CHASE stability pass deliberately filters the spatial target before
camera lag is applied.

1. Raw interpolated X/Y still projects onto the validated circuit centerline.
2. Projection continuity prevents jumps to nearby or crossing track branches.
3. Measured lap progress is converted into a one-dimensional presentation
   state with bounded velocity and bounded measurement correction.
4. Small backwards progress flicker is suppressed while the car is clearly
   travelling forwards. Large seeks still reset immediately.
5. The display path is a closed centripetal Catmull-Rom spline sampled densely
   from the validated centerline. The original centerline remains projection
   authority and is not replaced.
6. Filtered source-derived lateral offset is applied to the smooth road path
   for the car only. The ideal racing guide remains a separate presentation
   overlay and never defines vehicle position.
7. The car marker and camera consume the same filtered longitudinal progress,
   but the camera follows lateral movement through a slower dead-banded
   camera-only state. The car can therefore move naturally inside a small
   visual soft zone without dragging the entire rig with every line correction.
8. Historical position samples are Hermite reconstructed for CHASE so velocity
   direction remains continuous across source sample boundaries.
9. A short render-time prediction estimates the current measurement target,
   but rendered progress never snaps to it. A moving-target critically damped
   phase spring carries the car continuously across replay sample boundaries.
10. Camera position and aim use stored-velocity critically damped springs.
    Vertical damping is intentionally heavier than plan-view damping, matching
    the world-up/gimbal behavior used by mature third-person camera systems.
11. Real canonical speed/brake/throttle telemetry influences only camera
    presentation: braking gently compresses the fore-aft rig and strong
    throttle-plus-acceleration lets it release on corner exit. Driver progress
    itself is never slowed or accelerated for effect.
12. The car artwork is projected from the road Frenet frame, but its final
    screen basis is orthogonalized and longitudinal foreshortening is bounded.
    This preserves road-relative heading/pitch cues without letting a 2D SVG
    undergo the severe shear/stretch that a raw affine approximation of
    perspective can produce.
13. Chassis heading stays tangent to the path. Front steering follows the
    no-slip bicycle relation implied by local curvature; no drift/sideslip is
    synthesized.
14. The forward ideal guide begins ahead of the car and never renders under or
    behind the body.
15. Visible front-wheel steering is independently damped so local curvature
    noise cannot make the wheel articulation twitch even when the chassis path
    itself is smooth.
16. Only the local road corridor is projected at render frequency, reducing
    clipping churn and unnecessary geometry work.

This mirrors mature chase-camera architecture: first establish a stable target
transform, then apply position/aim damping to the camera. It also keeps the
presentation layer reversible because no timing or source telemetry product is
modified.

The spline uses centripetal parameterization rather than uniform Catmull-Rom.
That choice is consistent with Three.js's default Catmull-Rom curve type and
with the curve literature showing that centripetal parameterization avoids
local cusps and self-intersections within curve segments when control-point
spacing varies.

## CHASE scene and camera art direction

CHASE is a stylized telemetry stage, not a fabricated photorealistic circuit.
The scene may add depth, readability, and motion cues only when they preserve
that distinction.

The current scene is composed from independent layers:

1. dark world/sky field;
2. low-contrast race-identity ambient tint;
3. neutral horizon haze;
4. perspective ground reference grid;
5. speed-sensitive atmospheric field;
6. dark road foundation;
7. outer shoulder ribbon;
8. asphalt surface;
9. darker rubbered groove;
10. independent left/right road-edge definition;
11. distance-faded ideal-guide glow plus crisp guide stroke;
12. car/contact shadow, driver halo, corner callout, and existing HUD overlays.

The road receives atmospheric depth falloff so distant geometry loses contrast
into the horizon while nearby asphalt remains crisp. That creates depth without
inventing barriers, kerbs, buildings, vegetation, or terrain that the source
data does not provide.

Color has one responsibility per layer:

- driver/team color identifies the selected car;
- the ideal guide remains a restrained neutral cyan;
- race identity can tint the distant atmosphere at very low opacity;
- road materials remain neutral so the scene does not become a neon arcade
  treatment.

Speed can slightly increase horizon/ground/guide presence, but the effect must
remain subordinate to replay motion. Brake/throttle state is expressed through
the bounded camera rig rather than fake brake lights, speed lines, or color
flashes.

Do not add gear-shift exhaust flames, sparks, suspension movement, tyre smoke,
or other physical events unless Formula Vision has an authoritative source for
that event. A gear/RPM transition alone does not prove that a visible exhaust
flame occurred.

### Camera-input conditioning

The camera never consumes raw presentation inputs directly.

Before framing:

- selected-driver speed is low-pass/critically-damped into a camera-only speed
  signal;
- track curvature is dead-banded, bounded, and critically damped before it can
  affect camera side offset, height, arm length, or look-ahead;
- selected-driver lateral position has its own bounded critically-damped state;
- acceleration/braking is converted into a separate bounded fore-aft rig
  displacement.

This separation prevents one noisy curvature sample from simultaneously moving
the camera in four dimensions. It also keeps the car/road reconstruction
truthful while allowing the camera to feel heavier than the source data.

The acceleration/braking arm is intentionally asymmetric:

- strong acceleration can pull the camera roughly 1.65 m farther back;
- braking can gather the camera roughly 0.75 m toward the car;
- direction changes and engagement are deliberately damped rather than
  immediate;
- release is slower so the camera settles rather than pumps;
- the baseline speed response changes distance, height, and look-ahead together
  to preserve screen composition;
- FOV remains fixed, so the effect reads as physical rig motion rather than
  artificial zoom.

Position and aim are damped separately. Vertical response is deliberately
heavier than plan-view response so elevation/track noise does not create
visible bobbing. Curvature-driven lateral framing is also deliberately smaller
than the earlier CHASE pass; the next corner should be revealed by composition,
not by the camera swinging dramatically around the car.

## Camera dynamics

Raw GPS samples arrive much slower than the rendered 60 FPS experience.

The current CHASE camera treats the selected car/road point as a rigid anchor.
It damps only the camera rig offsets around that anchor:

- driver progress: filtered/projection-corrected, with canonical speed used for
  between-sample forward phase where available;
- chassis heading/pitch: measured across the vehicle wheelbase on the
  normalized road path;
- camera rig position and aim: critically damped in target-relative space;
- speed: defines the stable baseline chase distance and look-ahead;
- longitudinal acceleration/braking: adds only a small bounded transient
  fore-aft arm displacement;
- curvature: may widen the rig laterally and shorten look-ahead slightly;
- FOV: fixed;
- vertical camera variation: intentionally minimal;
- no direct camera response to single-sample GPS spikes.

The acceleration transient is deliberately asymmetric: strong throttle can
pull the camera back farther than braking pushes it forward. Its response is
fast enough to register on corner exit, but release is slower so it settles
without pumping.

All camera springs use replay-time delta rather than raw wall-clock delta. This
keeps camera character much more consistent across playback speeds and freezes
camera motion when replay is paused.

## External camera and rendering references

The CHASE polish pass is informed by mature camera/rendering systems rather than
copied visually from any one game:

- Gran Turismo 7 documents chase-camera sensitivity as a control over how
  slowly the camera responds to vehicle motion, and exposes rotation-center
  offset separately. That reinforces the goal of a heavier follow rig whose
  framing response is distinct from the vehicle transform:
  https://www.gran-turismo.com/us/gt7/manual/drivingoption/05
- Unity Cinemachine's Position Composer explicitly treats desired screen
  position, per-axis damping, look-ahead smoothing, dead-zone depth, and
  screen-space dead/soft zones as separate controls. Formula Vision mirrors
  that philosophy by keeping the car inside a tighter straight-line
  composition envelope while allowing bounded camera motion around it:
  https://docs.unity.cn/Packages/com.unity.cinemachine%406.6/manual/CinemachinePositionComposer.html
- Unity also notes that look-ahead on noisy target motion can amplify jitter;
  Formula Vision therefore conditions speed/progress before it can influence
  look-ahead or camera-arm response:
  https://docs.unity.cn/Packages/com.unity.cinemachine%403.1/manual/CinemachineRotationComposer.html
- Unreal Engine Spring Arm exposes position lag, rotation lag, a maximum lag
  distance, and lag sub-stepping as separate controls. That supports the same
  architecture here: bound the target/camera relationship first, then damp the
  camera rather than smoothing raw world motion indiscriminately:
  https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Engine/USpringArmComponent
- Turn 10 describes physically based lighting, volumetric atmosphere,
  wet/dry-surface response, and rubbering-in as major contributors to Forza
  Motorsport track richness. Formula Vision translates only the parts that can
  be represented truthfully from its data model, such as atmosphere, neutral
  road material hierarchy, and a rubbered groove:
  https://forza.net/news/forza-motorsport-developer-direct
- Driver61's racing-line material separates turn-in, apex, and exit and
  emphasizes using available track width; slower corners generally favor a
  later apex when exit speed matters more. Formula Vision uses that as a
  driver-facing geometry principle, not as a claim that the generated path is
  the mathematically fastest line:
  https://driver61.com/uni/racing-line/
- Driver61's sequence guidance is the critical exception to treating every
  corner independently: the exit of one corner is the entry to the next, so
  earlier elements of a complex are often compromised to position the car for
  the later, more important element:
  https://driver61.com/uni/different-corner-technique/
- Formula 1's own Abu Dhabi T12-13 trackside analysis describes exactly this
  behavior in an F1 car: drivers ran below the limit in the first half of the
  chicane so the car would be positioned correctly for the second. This is the
  real-world reason CHASE must relax full-width apex demands in close S-shapes:
  https://www.formula1.com/en/latest/article/trackside-performance-analysis-the-millimetric-challenge-of-abu-dhabis.3SywlbhTsa7rR4BMiuMXBa
- Heilmeier, Wischnewski, Hermansdorfer, Betz, Lienkamp, and Lohmann formulate
  minimum-curvature autonomous-racing trajectory generation as a constrained
  optimization problem. That is the technical ceiling Formula Vision's
  geometry heuristic is designed to approach conceptually without pretending
  to have the complete vehicle/track model:
  https://doi.org/10.1080/00423114.2019.1631455
- A 2026 Formula-1 telemetry study goes one step further and shows that
  expert-like lateral raceline offsets learned from real F1 trajectories make
  a better initialization for minimum-time trajectory optimization than a
  centerline or generic minimum-curvature seed. That points to the eventual
  highest-fidelity Formula Vision upgrade: precompute a representative
  telemetry-derived line and use the current geometry solver as the robust
  fallback:
  https://arxiv.org/abs/2603.07126
- Formula 1's own driver-conditioning material gives a useful reality check
  for the presentation envelope: sustained high-speed corner loads are
  commonly around 4-5 g, with peaks approaching 6 g and a documented 5.6 g
  peak at Mugello. Formula Vision uses 5.8 g only as a broad reconstruction
  guardrail, not as a tyre-model claim:
  https://www.formula1.com/en/latest/article/neck-workouts-cardio-sessions-and-mental-exercises-how-to-train-like-an-f1.6c5HZRQvQv2MALjFF295Aq
- Kapania, Subosits, and Gerdes explicitly separate racing trajectory
  generation into a speed-profile problem under tyre-friction constraints and
  a path-curvature optimization problem, iterating the two. That separation is
  directly relevant to Formula Vision's new geometry-intent versus
  time-domain-realization architecture:
  https://arxiv.org/abs/1902.00606
- Xue, Yue, and Dolan show that B-spline minimum-curvature optimization can
  provide mathematical smoothness guarantees even when sophisticated vehicle
  and track dynamics are unavailable. Formula Vision's deterministic
  reference-speed spatial regularizer is a much simpler browser-side analogue
  of the same principle:
  https://arxiv.org/abs/2309.09186
- Inverse-optimal-control racing research models observed trajectories as a
  tradeoff among progress, acceleration, jerk smoothness, and constraints. This
  supports explicitly limiting lateral jerk rather than treating positional
  smoothness alone as sufficient:
  https://arxiv.org/abs/2204.01494
- Gran Turismo and Toyota's Data Logger Visualization work is particularly
  relevant because it reconstructs recorded real driving inside a simulator.
  Their high-precision replay combines GPS with CAN channels including
  longitudinal/lateral g, steering, brake, accelerator, engine RPM, and
  individual wheel speeds. Formula Vision does not possess all of those
  channels, which is exactly why missing dynamics are bounded instead of
  fabricated:
  https://www.gran-turismo.com/us/news/02_0005189.html
- Gran Turismo Sophy's underlying simulator explicitly models tyre friction,
  aerodynamic resistance, and suspension-driven orientation changes. It is a
  useful reminder that believable grip emerges from coupled vehicle dynamics,
  not from a single racing-line spline:
  https://www.gran-turismo.com/us/gran-turismo-sophy/technology/
- R8 Games' Formula Fusion developers describe the sense of speed as strongly
  environment-based, explicitly calling out rapidly passing track texture.
  CHASE therefore uses sparse world-anchored asphalt detail for optical flow
  instead of screen-space speed lines:
  https://www.unrealengine.com/developer-interviews/how-formula-fusion-nailed-the-feeling-of-fast
- Criterion's Burnout work similarly relied on peripheral/environment detail
  to restore speed perception when road width increased. The relevant lesson
  is relative world motion and parallax, not copying Burnout's arcade effects:
  https://blog.playstation.com/archive/2017/08/31/classic-levels-deconstructed-burnout-3-takedowns-blistering-alpine-track
- Rockstar's current-generation GTAV release highlights improved motion blur
  and haptic response to rough road surfaces as part of richer vehicle motion
  perception. Formula Vision borrows the broader multisensory/relative-motion
  principle only; it does not add generic full-screen blur:
  https://www.rockstargames.com/newswire/article/172872k8a375k8/gtav-and-gta-online-coming-march-15-for-playstation-5-and-xbox-series
- Unreal's radial-motion-blur guidance exists specifically because ordinary
  screen-space blur is not sufficient for fast rotating objects such as
  wheels. That reinforces the decision not to fake wheel roll by simply
  rotating the current flat SVG tyre rectangles:
  https://dev.epicgames.com/documentation/en-us/unreal-engine/radial-motion-blur
- Unity's WheelCollider pose API treats wheel steering, rotation, suspension,
  and ground contact as a 3D wheel-pose problem. A future true wheel-roll
  upgrade should follow that model rather than a 2D decorative spin:
  https://docs.unity.com/en-us/engine/6000.5/script-reference/unityengine/wheelcollider/getworldpose

These references guide principles, not asset imitation. Formula Vision does not
invent scenery, surface state, exhaust flames, sparks, tyre smoke, or other
events that its source data cannot support.

## Track-material and racing-line visual hierarchy

The current visual target is not "more decoration"; it is stronger material
hierarchy and better ground attachment.

Best-in-class racing titles repeatedly converge on the same road-surface ideas:

- iRacing treats the visible groove as actual rubber laid onto the established
  racing line rather than as a generic dark center stripe. Its laser-scanned
  tracks also preserve cracks, pockmarks, camber, and undulation:
  https://www.iracing.com/track-technology/
  https://www.iracing.com/lets-talk-about-hot-rubber/
- Forza describes track realism as a combination of scanned/material-captured
  environments, physically based lighting, dynamic surface temperature,
  weather, and rubbering-in. The transferable lesson for Formula Vision is
  multi-scale surface response, not trying to imitate a photoreal texture with
  one flat fill:
  https://forza.net/news/forza-motorsport-the-most-technically-advanced-racing-game-ever-made
  https://forza.net/news/forza-motorsport-developer-direct
- Unreal's material guidance emphasizes roughness variation and decals for
  breaking repetition, grime, damage, and other local surface differences.
  Formula Vision maps that principle to deterministic world-space tonal
  patches, aggregate flecks, and seams:
  https://dev.epicgames.com/documentation/en-us/unreal-engine/physically-based-materials-in-unreal-engine
  https://dev.epicgames.com/documentation/en-us/unreal-engine/decal-materials-in-unreal-engine
- Gran Turismo's driving-line assist deliberately uses a restrained dotted line,
  while F1 24/25 exposes both 2D/3D dynamic racing-line modes and adjustable
  opacity. The useful design principle is that guidance should read as a
  ground-attached aid with clear depth, not as a flat HUD polyline:
  https://www.gran-turismo.com/au/gt7/manual/drivingoption/03
  https://www.ea.com/able/resources/f1-25

That leads to the following rendering stack, from foundational to decorative:

1. **Road silhouette and shoulder width** define the physical track envelope.
2. **Base asphalt tone** carries distance/atmospheric falloff.
3. **Subtle cross-road sheen** gives the otherwise-flat SVG surface a broad
   roughness/light-response cue without pretending to be a PBR shader.
4. **World-anchored resurfacing patches, seams, and aggregate flecks** break
   repetition at multiple spatial scales and create real optical flow.
5. **Presentation rubber groove** follows the normalized racing line rather than
   the circuit centerline. A broad low-contrast groove is reinforced by two
   faint tyre-path ribbons offset from the line.
6. **Racing-line guidance** is rendered as small world-space ribbon segments.
   The dash length and width are measured in meters before projection, so they
   naturally shrink with distance instead of remaining constant screen-space
   dashes at the horizon.
7. **Car/contact shadow** remains the top ground-contact layer.

The rubber groove is explicitly a **representative presentation layer**, not a
claim that Formula Vision has simulated session-specific rubber deposition.
Its job is to visually teach the road's preferred line and make the generated
trajectory feel embedded in the asphalt.

The background is intentionally *not* being expanded into trees, barriers,
grandstands, kerbs, runoff paint, or circuit furniture in this pass. Those
objects are powerful parallax/depth cues in Forza, Gran Turismo, iRacing, and
other titles, but Formula Vision currently lacks authoritative geometry for
them. Inventing generic scenery would increase visual density while decreasing
circuit truth. Until circuit-side geometry exists, the background should remain
an abstract atmospheric stage: horizon separation, subtle ground depth, race
theme tint, and vignette only.

That sequencing is deliberate: road material, line grounding, vehicle physics,
and camera motion must be convincing before niche scenery or decorative effects
are allowed to compete for attention.

## Ground interaction and wheel rotation policy

The first requirement for believable motion is **relative ground motion**.
Keeping the followed car compositionally stable is desirable for a chase
camera, but it removes one of the easiest visual speed cues: the car crossing
the screen. CHASE now restores that cue in the road itself.

The asphalt renderer adds two sparse deterministic world-space layers:

- short irregular grain/fleck segments sampled every ~8 m within the road
  surface;
- rare low-contrast resurfacing seams sampled roughly every ~46 m.

Both are generated from global circuit distance and projected through the same
camera/road transform as the asphalt. They therefore move under the vehicle at
the replay's actual spatial rate, accelerate and decelerate with the scene,
respect perspective, stay fixed to the world through camera motion, and freeze
when replay is paused. Their opacity rises only slightly with speed. This is
deliberately different from screen-space speed lines, particles, or a moving
texture UV trick, all of which can make the road look detached from the world.

The car also has four tyre-local contact shadows plus visible suspension links.
Those additions improve the open-wheel silhouette and make the tyres read as
four separate contact patches rather than rectangles floating beside a body.

Formula Vision still does **not** fake tyre rolling by rotating the existing SVG
tyre rectangles. Real wheel roll is rotation about the vehicle's lateral axle;
a normal SVG `rotate()` would spin the footprint around the road-normal axis
and would be physically wrong. High angular velocity also creates temporal
aliasing and normally needs a rotation-aware blur/shader solution.

The available speed channel would be enough to compute a no-slip angular
velocity with `omega = v / r`, but visually using that phase should wait for a
small projected 3D wheel/sidewall model (or a full 3D vehicle). Likewise,
constant tyre trails, smoke, or skid marks are intentionally rejected because
they imply wheel slip or rubber deposition that Formula Vision does not have an
authoritative channel to prove. Front-wheel steering remains rendered from the
no-slip bicycle model because path curvature is available and that transform is
representable correctly in the current road plane.

## Handling difficult circuits and edge cases

### Crossings / nearby segments

Use continuity-aware projection to avoid jumping across the circuit.

### Start/finish wrap

Arc length is circular. Camera orientation must interpolate across `s = length -> 0` without discontinuity.

### Pit lane

Current track geometry is the race centerline, not an authoritative pit-lane map.

Initial behavior:
- selected driver remains spatially visible from raw position
- camera can follow raw XY while preserving last stable track tangent
- do not falsely snap a pit-lane car onto the racing line

Later:
- derive or publish a pit-lane branch polyline.

### Off-track / crashes

Allow raw lateral position away from the centerline.
The track frame still gives orientation and nearby spatial context.

### Retirements / missing spatial data

Freeze briefly, then release camera to whole-track or timing-led state.
Never fabricate continued motion.

### Seeks

Reset smoothing and permit one global projection search so the camera immediately relocates to the correct part of the circuit.

### Red flags / restarts

Camera system follows replay timestamps and positions; race state does not need special spatial logic except optional cinematic framing.

## Elevation

FastF1 canonical telemetry already stores Z.

Use three levels:

### Level A: flat perspective

Always available when X/Y exists.
The track is physically 3D but lies on a plane.
This already produces the elevated/tilted map feeling.

### Level B: source elevation

Use Z when:

- enough track samples have finite Z
- the vertical range is plausible
- discontinuities are within quality thresholds

Smooth elevation aggressively because camera motion exaggerates noise.

### Level C: external terrain

Not required and intentionally out of scope for the first 3D release.
It would introduce geographic alignment, licensing, terrain-source, and accuracy complexity without improving the core Formula Vision idea enough.

## Data-derived visual layers

The 3D engine unlocks more than a camera.

### Speed surface

Aggregate driver speed by circuit progress and color the track ribbon or a thin overlay.

### Braking map

Aggregate brake activation by `s`.
Display braking zones as subtle red edge segments.

### Throttle map

Show acceleration zones or live selected-driver throttle as a wake/trail.

### Gear / RPM ribbon

Optional analysis lens that paints average gear/RPM around the circuit.

### DRS activation map

Derive regions where DRS activation is consistently observed.
Label these as observed activation regions unless authoritative zone metadata is available.

### Tyre/strategy traces

Render stint segments or selected-lap trails with compound color.

### Incident markers

Race-control events can anchor to the selected/relevant driver's spatial position when enough context exists.

### Overtake reconstruction

Timing position change + spatial proximity + race-control/lap context can surface likely overtaking moments.
Do not label inferred passes as authoritative without sufficient evidence.

## Circuit metadata

Circuit metadata is implemented as an optional additive canonical/browser
contract.

Canonical extraction attempts FastF1 `get_circuit_info()` and writes
`circuit_info.json` when available. The browser builder then publishes the
same data under `manifest.metadata.circuit`.

A circuit-info source failure is warning-only. Race timing, replay, MAP, and
CHASE remain valid. Existing releases without the field use frontend geometric
corner inference.

Historical releases from 2018 onward can be enriched by deliberately rebuilding
canonical and browser artifacts and publishing new immutable releases. This
requires source reacquisition because old canonical snapshots did not store
FastF1 circuit-info responses. Elevation-only rebuilds remain different:
existing canonical position telemetry already preserves Z and does not require
source reacquisition solely for elevation.

## Browser artifact evolution

Current browser position chunks publish X/Y.

Future backward-compatible spatial release can expose:

- optional Z in position chunks
- 3D track points
- track length
- elevation quality
- arc-length lookup samples
- curvature
- optional multiple geometry LODs

Old immutable releases remain readable.
The frontend should feature-detect richer geometry.

Existing canonical snapshots already preserve Z, so old races can be upgraded by rebuilding browser artifacts and publishing new immutable releases. No FastF1 reacquisition is required solely for elevation.

## Performance

3D is opt-in or capability-adaptive, not mandatory for landing-page rendering.

Rules:

- lazy-load the 3D bundle when a 3D camera is selected
- keep normal race library free of Three.js cost
- use instanced driver markers
- reuse geometry/materials
- cap DPR adaptively
- scale effects down under sustained frame pressure
- avoid heavyweight postprocessing initially
- pause/demand-render when replay is paused and scene is static
- preserve current SVG map as a fallback

React Three Fiber documents adaptive DPR/performance controls and on-demand rendering, which fits this architecture.

## Product UI

Viewer gets a compact camera selector:

`MAP | FOLLOW | CHASE | HELI`

Do not expose every experimental mode at launch.

When a driver is selected:
- FOLLOW remains analytical
- CHASE becomes immersive
- HELI becomes contextual/spatial

Camera mode changes should preserve the same selected driver and replay timestamp.

A small temporary label can identify mode, then fade.

## Implementation phases

### S0: math foundation

- optional Z-ready frontend types
- universal spatial track frame
- continuity-aware track projection
- generic camera-pose presets
- frame-rate independent camera smoothing

Status: foundation implemented in `frontend/src/lib/spatialTrack.ts`.

### S1: flat 3D prototype

- add Three.js/R3F as lazy-loaded viewer dependency
- render track ribbon using current X/Y
- render all drivers as simple instanced markers
- add orbit/inspection camera for development
- prove Monaco + Suzuka + Spa + one street circuit

### S2: selected-driver camera

- CHASE camera
- HELI camera
- seek/reset handling
- pit/off-track fallback
- camera mode selector
- reduced-motion/static fallback

### S3: elevation

- expose optional Z from browser artifacts
- compute elevation quality
- rebuild a small hostile-fixture set
- enable source elevation only for quality-passing races

### S4: circuit intelligence

- curvature and auto-corner detection
- optional FastF1 CircuitInfo acquisition
- corner labels
- braking/throttle/speed spatial overlays

### S5: cinematic/broadcast system

- geometry-derived virtual camera nodes
- camera cuts with minimum shot duration
- selected-driver and leader modes
- replay-event-aware cuts

### S6: DUEL and analysis lenses

- two-driver dynamic camera
- synchronized telemetry comparison
- gap visualization
- lap ghost/trails
- strategy spatial overlays

## Definition of success

The 3D feature is successful when:

1. the same code works on every validated circuit without hand-authored camera paths;
2. a driver can be followed through every corner without camera snapping;
3. Suzuka-like geometry does not cause branch jumps;
4. pit/off-track/missing-data behavior fails gracefully;
5. the feature remains smooth on a normal laptop;
6. flat-X/Y races still look intentional;
7. true elevation enhances good data without corrupting poor data;
8. switching back to the existing map loses no replay state;
9. timing remains authoritative even though the spatial presentation is cinematic;
10. the resulting experience could only plausibly belong to Formula Vision.


## Additive camera acceptance checklist

Before making CHASE a default or replacing any existing camera behavior, verify all of the following on the deployed preview:

- [ ] Race opens in MAP exactly as before.
- [ ] Existing top-down driver follow still works exactly as before.
- [ ] Selecting CHASE while already following a driver follows the same driver.
- [ ] Selecting CHASE with no follow target acquires a sensible current target.
- [ ] CHASE visibly uses perspective: near track is wider/larger and distant track recedes.
- [ ] Followed car remains compositionally stable through slow and fast corners.
- [ ] Motion stays continuous when the camera RAF renders between replay-frame updates.
- [ ] Source sample boundaries do not produce visible direction/velocity kinks.
- [ ] Constant forward progress does not visibly pulse in speed within a curved spline segment.
- [ ] Track edges remain visually stationary in world space while the car moves through them.
- [ ] The road surface continues beneath and behind the camera without a visible
  rear cutoff as old samples leave the local corridor.
- [ ] Car remains visually planted on the road through elevation changes; it
  never appears to hover, hop, or be pulled vertically.
- [ ] Chassis and tyres form one rigid road-plane transform with front-wheel
  steering only; no synthetic body sideslip is visible.
- [ ] On an isolated normal corner, CHASE visibly sets up from the outside,
  clips the inside near the apex, and tracks back toward the outside on exit.
- [ ] Close opposite-direction corners transition smoothly between clipping
  points without an artificial full-width left-right oscillation.
- [ ] A short S/chicane never demands more lateral travel than its available
  longitudinal distance can support; the synthesized cross-track angle stays
  bounded instead of producing a visible dart from one side of the road to the
  other.
- [ ] At 250-330 km/h, a left-right or right-left sequence spreads the
  cross-track move over substantially more race time/distance than the same
  nominal offset change at hairpin speed.
- [ ] Increasing playback from 1x to 4x makes the same reconstructed lateral
  trajectory play four times faster; it does not change the physical shape of
  that trajectory.
- [ ] Speed, RPM, and throttle no longer visibly change only at source packet
  boundaries when adjacent historical samples are available.
- [ ] Lateral velocity, lateral acceleration, and lateral jerk remain bounded
  through a target sign reversal; the car cannot instantaneously inherit the
  opposite apex tangent.
- [ ] Strong braking or a high-curvature/high-speed combination reduces spare
  synthetic lateral authority instead of allowing an unconstrained extra
  cross-track correction.
- [ ] The chase camera follows/extrapolates realized lateral motion and never
  swings toward a geometric target faster than the car itself can get there.
- [ ] The same isolated corner tested before this pass still uses essentially
  the same outside setup, apex depth, and track-out when it is separated by a
  meaningful straight.
- [ ] Tighter corners can apex slightly later than fast bends while the path
  remains continuous and inside the rendered road envelope.
- [ ] Front-wheel steering direction/magnitude matches the generated vehicle
  path curvature rather than a separate centerline approximation.
- [ ] Vehicle heading/pitch remains stable across tiny local tangent changes
  because orientation is evaluated across the wheelbase.
- [ ] The forward guide starts ahead of the car and never visually passes
  through or behind the car body.
- [ ] During normal on-track running, only a small bounded source-lateral
  residue perturbs the normalized racing line; authoritative progress remains
  source-derived.
- [ ] Clearly off-line/pit/incident source positions fall back to the full
  filtered lateral displacement instead of being forced back onto the racing
  line.
- [ ] Racing-line geometry never changes authoritative progress, timing, camera
  ownership, or source telemetry.
- [ ] MAP corner labels reveal progressively with zoom and reject label-label
  overlap.
- [ ] MAP corner labels disappear rather than cover a visible driver marker.
- [ ] CHASE shows at most one contextual corner callout in a side safe zone and
  never places corner text over the road/car.
- [ ] Published FastF1 turn numbers are used when circuit metadata is present;
  older releases fall back safely to geometric turn inference.
- [ ] Named corners come only from the curated common-name mapping; unsupported
  names remain numbered turns.
- [ ] Pausing freezes CHASE without presentation drift.
- [ ] Braking presentation is visible but subtle and never changes authoritative
  driver progress.
- [ ] Strong-throttle corner exits feel more energetic without zoom pumping or
  arcade-style speed-line effects.
- [ ] 0.5x, 1x, 2x, and 4x playback preserve the same qualitative camera feel.
- [ ] Pausing also freezes camera spring/inertia state visually.
- [ ] Strong acceleration produces a visible but restrained temporary camera
  pull-back without FOV pumping or vertical movement.
- [ ] Braking produces a smaller smooth temporary camera gather toward the car.
- [ ] Curvature sign changes on straights do not make the rig twitch laterally.
- [ ] Camera speed/curvature conditioning removes sample-driven framing chatter
  without making steering response feel delayed.
- [ ] Distant asphalt loses contrast into the horizon while the near road
  remains crisp.
- [ ] The broad rubber groove follows the normalized racing line rather than the
  geometric centerline, with faint tyre-path bands nested inside it.
- [ ] Racing-line dash length and width shrink naturally with perspective
  because segments are authored in world meters, not fixed screen pixels.
- [ ] Surface patches, seams, and aggregate remain fixed to circuit world space
  and never swim under camera motion.
- [ ] Shoulder, surface, groove, and road edges remain visually distinct without
  reading as invented kerbs or lane markings.
- [ ] Race ambient color remains subtle enough that team color and guide color
  keep separate semantic roles.
- [ ] Scene-speed response adds depth without visible speed lines, shake, or
  decorative particle effects.
- [ ] Asphalt grain and sparse seams remain anchored to circuit world space,
  flow underneath the car in perspective, and freeze exactly when replay is
  paused.
- [ ] Surface detail does not visibly slide sideways with camera lag or read as
  lane markings.
- [ ] Tyre contact patches stay aligned under all four wheels through steering
  and corner projection; no synthetic skid/smoke trail appears during ordinary
  grip driving.
- [ ] Neither effect changes authoritative driver progress.
- [ ] Normal forward running never shows small backwards marker flicker.
- [ ] Marker heading does not snap when projected angle crosses -180/180 degrees.
- [ ] Forward trace advances continuously rather than snapping between samples.
- [ ] Fast straights pull the camera back/look farther ahead without abrupt FOV-like jumps.
- [ ] Tight corners shorten the view and offset subtly to reveal the corner.
- [ ] Timing-tower selection switches the CHASE target without dropping back to MAP.
- [ ] Switching back to MAP returns to the same driver and same replay timestamp.
- [ ] Play, pause, playback speed, scrub, and ±10 second seeking work in both modes.
- [ ] A large seek relocates CHASE immediately rather than flying across the map.
- [ ] Start/finish wrap does not flip the camera.
- [ ] Suzuka / nearby track branches do not cause projection jumps.
- [ ] Pit-lane / off-line spatial samples do not crash the renderer.
- [ ] Driver retirement / missing position data fails back safely if the target cannot be resolved.
- [ ] Driver inspector, race control, results, timeline events, and timing authority are unchanged.
- [ ] Mobile/narrow layouts remain usable; camera selector does not cover core controls.
- [ ] Reduced-motion preference does not introduce continuous decorative camera/UI animation outside replay motion.
- [ ] Returning to MAP after any CHASE issue restores the proven renderer without reloading the race.

